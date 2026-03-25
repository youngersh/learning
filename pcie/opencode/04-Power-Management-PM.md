# Power Management (PM) Capability 详细分析

## 概述

**Capability ID**: PCI_CAP_ID_PM (0x01)  
**规范参考**: PCI Power Management Interface Specification 1.2+  
**功能**: 支持设备电源状态管理（D0-D3cold）和 PME# 唤醒

## PCIe SPEC 定义

### Capability 结构
```
Offset  Size  Description
0x00    1    Capability ID (0x01)
0x01    1    Next Capability Pointer
0x02    2    Capability (PMC)
              - Bits [2:0]: Version
              - Bit 3: PME Clock
              - Bit 5: DSI
              - Bits [7:6]: Aux Power
              - Bit 8: D1 Support
              - Bit 10: D2 Support
              - Bit 11: PME Support (D0)
              - Bit 12: PME Support (D1)
              - Bit 13: PME Support (D2)
              - Bit 14: PME Support (D3hot)
              - Bit 15: PME Support (D3cold)
0x04    2    Control/Status (PMCSR)
              - Bits [1:0]: Power State
              - Bit 3: No Soft Reset
              - Bit 8: PME Enable
              - Bits [13:10]: Data Select
              - Bits [15:14]: Data Scale
0x06    1    Bridge Extensions (Type 1 only)
0x07    1    Data (Data Register)
```

### Power States
| 状态 | 描述 |
|------|------|
| D0 | 全功率 |
| D1 | 低功率，快速恢复 |
| D2 | 更低功率，较慢恢复 |
| D3hot | 软关闭，保持供电 |
| D3cold | 软关闭，移除供电 |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/pci.c`  
**函数**: `pci_pm_init()` (行 3133-3201)

```c
void pci_pm_init(struct pci_dev *dev)
{
    int pm;
    u16 pmc;

    // 1. 启用异步挂起
    device_enable_async_suspend(&dev->dev);
    dev->wakeup_prepared = false;

    dev->pm_cap = 0;
    dev->pme_support = 0;

    // 2. 查找 PM capability
    pm = pci_find_capability(dev, PCI_CAP_ID_PM);
    if (!pm)
        goto poweron;

    // 3. 读取 PMC
    pci_read_config_word(dev, pm + PCI_PM_PMC, &pmc);

    // 4. 检查版本
    if ((pmc & PCI_PM_CAP_VER_MASK) > 3) {
        pci_err(dev, "unsupported PM cap regs version (%u)\n",
            pmc & PCI_PM_CAP_VER_MASK);
        goto poweron;
    }

    dev->pm_cap = pm;
    dev->d3hot_delay = PCI_PM_D3HOT_WAIT;  // 10ms
    dev->d3cold_delay = PCI_PM_D3COLD_WAIT; // 100ms
    dev->bridge_d3 = pci_bridge_d3_possible(dev);
    dev->d3cold_allowed = true;

    dev->d1_support = false;
    dev->d2_support = false;

    // 5. 检查 D1/D2 支持
    if (!pci_no_d1d2(dev)) {
        if (pmc & PCI_PM_CAP_D1)
            dev->d1_support = true;
        if (pmc & PCI_PM_CAP_D2)
            dev->d2_support = true;

        if (dev->d1_support || dev->d2_support)
            pci_info(dev, "supports%s%s\n",
                   dev->d1_support ? " D1" : "",
                   dev->d2_support ? " D2" : "");
    }

    // 6. 检查 PME 支持
    pmc &= PCI_PM_CAP_PME_MASK;
    if (pmc) {
        pci_info(dev, "PME# supported from%s%s%s%s\n",
                 (pmc & PCI_PM_CAP_PME_D0) ? " D0" : "",
                 (pmc & PCI_PM_CAP_PME_D1) ? " D1" : "",
                 (pmc & PCI_PM_CAP_PME_D2) ? " D2" : "",
                 (pmc & PCI_PM_CAP_PME_D3hot) ? " D3hot" : "",
                 (pmc & PCI_PM_CAP_PME_D3cold) ? " D3cold" : "");
        dev->pme_support = FIELD_GET(PCI_PM_CAP_PME_MASK, pmc);
        dev->pme_poll = true;

        // 设置唤醒能力
        device_set_wakeup_capable(&dev->dev, true);

        // 禁用 PME#
        pci_pme_active(dev, false);
    }

poweron:
    // 7. 上电到 D0
    pci_pm_power_up_and_verify_state(dev);

    // 8. 配置运行时电源管理
    pm_runtime_forbid(&dev->dev);
    pm_runtime_set_active(&dev->dev);
    pm_runtime_enable(&dev->dev);
}
```

### 电源状态转换

#### 1. pci_set_power_state()
```c
int pci_set_power_state(struct pci_dev *dev, pci_power_t state)
{
    u16 pmcsr;

    if (!dev->pm_cap)
        return -EINVAL;

    // 1. 读取当前状态
    pci_read_config_word(dev, dev->pm_cap + PCI_PM_CTRL, &pmcsr);

    // 2. 设置新状态
    pmcsr &= ~PCI_PM_CTRL_STATE_MASK;
    pmcsr |= state;

    // 3. 写入配置空间
    pci_write_config_word(dev, dev->pm_cap + PCI_PM_CTRL, pmcsr);

    // 4. 更新设备状态
    dev->current_state = state;

    return 0;
}
```

#### 2. pci_choose_state()
```c
pci_power_t pci_choose_state(struct pci_dev *pdev)
{
    // 优先使用 ACPI 决策
    if (acpi_pci_power_manageable(pdev))
        return acpi_pci_choose_state(pdev);

    // 默认策略
    if (pdev->pme_support)
        return PCI_D3hot;

    return PCI_D0;
}
```

#### 3. pci_power_up()
```c
int pci_power_up(struct pci_dev *dev)
{
    int state;

    // 1. 检查是否需要上电
    state = pci_raw_get_power_state(dev);
    if (state == PCI_D0)
        return 0;

    // 2. 设置为 D0
    pci_set_power_state(dev, PCI_D0);

    // 3. 等待稳定
    if (state == PCI_D3hot || state == PCI_D3cold) {
        pci_restore_bars(dev);
        msleep(dev->d3hot_delay);
    }

    return 0;
}
```

### PME (Power Management Event) 处理

#### 1. pci_pme_active()
```c
void pci_pme_active(struct pci_dev *dev, bool enable)
{
    u16 pmcsr;

    if (!dev->pm_cap)
        return;

    pci_read_config_word(dev, dev->pm_cap + PCI_PM_CTRL, &pmcsr);

    if (enable)
        pmcsr |= PCI_PM_CTRL_PME_ENABLE;
    else
        pmcsr &= ~PCI_PM_CTRL_PME_ENABLE;

    pci_write_config_word(dev, dev->pm_cap + PCI_PM_CTRL, pmcsr);
}
```

#### 2. pci_pme_wakeup()
```c
bool pci_pme_wakeup(struct pci_dev *dev)
{
    u16 pmcsr;

    if (!dev->pm_cap)
        return false;

    pci_read_config_word(dev, dev->pm_cap + PCI_PM_CTRL, &pmcsr);

    return (pmcsr & PCI_PM_CTRL_PME_STATUS) != 0;
}
```

#### 3. pci_check_pme_status()
```c
bool pci_check_pme_status(struct pci_dev *dev)
{
    u16 pmcsr;

    if (!dev->pm_cap)
        return false;

    pci_read_config_word(dev, dev->pm_cap + PCI_PM_CTRL, &pmcsr);

    if (!(pmcsr & PCI_PM_CTRL_PME_STATUS))
        return false;

    // 清除 PME 状态
    pci_write_config_word(dev, dev->pm_cap + PCI_PM_CTRL,
                      pmcsr | PCI_PM_CTRL_PME_STATUS);

    return true;
}
```

### 运行时电源管理

#### 1. pci_runtime_suspend()
```c
int pci_runtime_suspend(struct pci_dev *pdev)
{
    pci_power_t state;

    // 1. 选择目标状态
    state = pci_choose_state(pdev);

    // 2. 转换到目标状态
    pci_set_power_state(pdev, state);

    // 3. 如果是 D3cold，完全断电
    if (state == PCI_D3cold) {
        pci_dev_fully_d3(pdev);
        pci_set_power_state(pdev, PCI_D3cold);
    }

    return 0;
}
```

#### 2. pci_runtime_resume()
```c
int pci_runtime_resume(struct pci_dev *pdev)
{
    // 1. 上电到 D0
    pci_power_up(pdev);

    // 2. 恢复配置
    pci_restore_state(pdev);

    return 0;
}
```

## Root Port 处理

Root Port 通常支持 PM capability，用于管理下游设备的电源状态：

```c
// 检查 Root Port 的 D3 支持
if (pci_pcie_type(dev) == PCI_EXP_TYPE_ROOT_PORT) {
    if (dev->bridge_d3) {
        // Root Port 可以进入 D3
        pci_info(dev, "Root Port supports D3\n");
    }
}
```

## 设备驱动使用

### 1. 设置电源状态
```c
// 进入 D3hot
pci_set_power_state(pdev, PCI_D3hot);

// 恢复到 D0
pci_set_power_state(pdev, PCI_D0);
```

### 2. 运行时电源管理
```c
// 启用运行时电源管理
pm_runtime_allow(&pdev->dev);

// 手动挂起
pm_runtime_suspend(&pdev->dev);

// 手动恢复
pm_runtime_resume(&pdev->dev);
```

### 3. PME 唤醒
```c
// 启用 PME
device_set_wakeup_enable(&pdev->dev, true);

// 检查 PME 状态
if (pci_check_pme_status(pdev)) {
    // 处理唤醒事件
}
```

### 4. 检查电源状态
```c
pci_power_t state = pci_get_power_state(pdev);

switch (state) {
case PCI_D0:
    // 设备全功率
    break;
case PCI_D3hot:
case PCI_D3cold:
    // 设备低功耗
    break;
}
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_pm_init()
                 ├─> pci_find_capability(PCI_CAP_ID_PM)
                 ├─> pci_read_config_word()  // 读取 PMC
                 ├─> device_set_wakeup_capable()  // 设置 PME 能力
                 ├─> pci_pme_active()  // 禁用 PME
                 ├─> pci_pm_power_up_and_verify_state()  // 上电到 D0
                 └─> pm_runtime_enable()  // 启用运行时 PM

驱动调用:
  └─> pci_set_power_state()
       └─> pci_write_config_word()  // 写入 PMCSR
  └─> pci_runtime_suspend()
       ├─> pci_choose_state()  // 选择目标状态
       └─> pci_set_power_state()  // 转换状态
```

## 配置空间访问

### 查找 PM Capability
```c
int pm_pos = pci_find_capability(dev, PCI_CAP_ID_PM);
```

### 读取 PMC
```c
u16 pmc;
pci_read_config_word(dev, pm_pos + PCI_PM_PMC, &pmc);

// 检查版本
int version = pmc & PCI_PM_CAP_VER_MASK;

// 检查 D1/D2 支持
bool d1_support = (pmc & PCI_PM_CAP_D1) != 0;
bool d2_support = (pmc & PCI_PM_CAP_D2) != 0;

// 检查 PME 支持
bool pme_support = (pmc & PCI_PM_CAP_PME_MASK) != 0;
```

### 读取/写入 PMCSR
```c
u16 pmcsr;

// 读取当前状态
pci_read_config_word(dev, pm_pos + PCI_PM_CTRL, &pmcsr);

// 设置新状态
pmcsr &= ~PCI_PM_CTRL_STATE_MASK;
pmcsr |= PCI_D3hot;

// 写入配置空间
pci_write_config_word(dev, pm_pos + PCI_PM_CTRL, pmcsr);
```

### PME 控制
```c
u16 pmcsr;
pci_read_config_word(dev, pm_pos + PCI_PM_CTRL, &pmcsr);

// 启用 PME
pmcsr |= PCI_PM_CTRL_PME_ENABLE;

// 清除 PME 状态
pmcsr |= PCI_PM_CTRL_PME_STATUS;

pci_write_config_word(dev, pm_pos + PCI_PM_CTRL, pmcsr);
```

## 电源状态转换图

```
D0 ←→ D1 ←→ D2 ←→ D3hot ←→ D3cold
 ↑                                      ↓
 └────────────────────────────────────────┘
```

转换规则：
- D0 → D1/D2/D3hot/D3cold: 直接转换
- D1/D2 → D0: 快速恢复
- D3hot → D0: 需要 d3hot_delay 延迟
- D3cold → D0: 需要 d3cold_delay 延迟，可能需要重新初始化

## PME 处理流程

```
设备进入低功耗
    ↓
PME# 信号断言
    ↓
系统唤醒
    ↓
驱动检查 PME 状态
    ↓
处理唤醒事件
    ↓
设备恢复到 D0
```

## 性能考虑

1. **D3cold 延迟**: 从 D3cold 恢复需要较长时间（100ms+）
2. **PME 轮询**: 需要定期检查 PME 状态
3. **运行时 PM**: 允许设备在空闲时自动进入低功耗
4. **Bridge D3**: Bridge 可以进入 D3，影响所有下游设备

## 调试信息

```c
// 在 pci_pm_init() 中
pci_info(dev, "supports%s%s\n",
         dev->d1_support ? " D1" : "",
         dev->d2_support ? " D2" : "");

pci_info(dev, "PME# supported from%s%s%s%s\n",
         (pmc & PCI_PM_CAP_PME_D0) ? " D0" : "",
         (pmc & PCI_PM_CAP_PME_D1) ? " D1" : "",
         (pmc & PCI_PM_CAP_PME_D2) ? " D2" : "",
         (pmc & PCI_PM_CAP_PME_D3hot) ? " D3hot" : "",
         (pmc & PCI_PM_CAP_PME_D3cold) ? " D3cold" : "");
```

## 错误处理

1. **PM capability 不存在**:
   ```c
   if (!dev->pm_cap)
       goto poweron;  // 直接上电
   ```

2. **版本不支持**:
   ```c
   if ((pmc & PCI_PM_CAP_VER_MASK) > 3) {
       pci_err(dev, "unsupported PM cap regs version (%u)\n",
           pmc & PCI_PM_CAP_VER_MASK);
       goto poweron;
   }
   ```

3. **状态转换失败**:
   ```c
   ret = pci_set_power_state(dev, PCI_D3hot);
   if (ret) {
       pci_err(dev, "Failed to set power state: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PM`: 启用电源管理
- `CONFIG_PM_SLEEP`: 支持系统睡眠
- `CONFIG_PM_RUNTIME`: 支持运行时电源管理
- `CONFIG_ACPI`: ACPI 电源管理

## 参考资料

- PCI Bus Power Management Interface Specification 1.2
- PCIe Base Specification Revision 5.0+, Section 5.9
- Linux 内核源码: `drivers/pci/pci.c`
- 头文件: `include/linux/pm.h`, `include/uapi/linux/pci_regs.h`
