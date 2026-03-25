# Downstream Port Containment (DPC) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_DPC (0x1D)  
**规范参考**: PCIe DPC Specification  
**功能**: 在 Downstream Port 检测和隔离错误，防止错误传播

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x1D)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    2    DPC Capability
              - Bit 0: DPC Trigger Enable
              - Bit 1: DPC Completion Enable
              - Bit 2: Interrupt Generation
              - Bit 3: RP Extensions
              - Bit 4: Poisoned TLP Blocking Enable
              - Bit 5: Swizzle Enable
              - Bit 6: Swizzle Completion Enable
              - Bit[11:8]: RP PIO Log Size
0x06    2    DPC Control
              - Bit 0: DPC Trigger Enable
              - Bit 1: DPC Trigger Reason
              - Bit 2: DPC Completion Enable
              - Bit 3: DPC Completion Reason
              - Bit 4: Interrupt Enable
0x08    2    DPC Status
              - Bit 0: DPC Trigger Status
              - Bit 1: DPC Completion Status
              - Bit 2: Interrupt Status
0x0A    2    DPC Reason
0x0C    N    DPC Error Source Identifier
0x10    N    DPC RP PIO Log
```

### DPC Trigger Reason
| 值 | 原因 |
|-----|------|
| 0 | Uncorrectable Error |
| 1 | Poisoned TLP |
| 2 | Flow Control Protocol Error |
| 3 | Completion Timeout |

### DPC Completion Reason
| 值 | 原因 |
|-----|------|
| 0 | Completion Timeout |
| 1 | Completion Abort |
| 2 | Unexpected Completion |
| 3 | Receiver Overflow |
| 4 | Bad TLP |
| 5 | Bad DLLP |
| 6 | Replay Number Rollover |
| 7 | Replay Timer Timeout |
| 8 | Advisory Non-Fatal Error |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/pcie/dpc.c`  
**函数**: `pci_dpc_init()` (行 400-436)

```c
void pci_dpc_init(struct pci_dev *pdev)
{
    u16 cap;

    // 1. 查找 DPC capability
    pdev->dpc_cap = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_DPC);
    if (!pdev->dpc_cap)
        return;

    // 2. 读取 DPC Capability
    pci_read_config_word(pdev, pdev->dpc_cap + PCI_EXP_DPC_CAP, &cap);
    if (!(cap & PCI_EXP_DPC_CAP_RP_EXT))
        return;

    // 3. 标记为 RP Extensions
    pdev->dpc_rp_extensions = true;

    // 4. 配置 RP PIO Log Size
    if (!pdev->dpc_rp_log_size) {
        u16 flags;
        int ret;

        ret = pcie_capability_read_word(pdev, PCI_EXP_FLAGS, &flags);
        if (ret)
            return;

        pdev->dpc_rp_log_size =
                FIELD_GET(PCI_EXP_DPC_RP_PIO_LOG_SIZE, cap);
        if (FIELD_GET(PCI_EXP_FLAGS_FLIT, flags))
            pdev->dpc_rp_log_size += FIELD_GET(PCI_EXP_DPC_RP_PIO_LOG_SIZE4,
                                                   cap) << 4;

        if (pdev->dpc_rp_log_size < PCIE_STD_NUM_TLP_HEADERLOG ||
            pdev->dpc_rp_log_size > PCIE_STD_MAX_TLP_HEADERLOG + 1) {
            pci_err(pdev, "RP PIO log size %u is invalid\n",
                     pdev->dpc_rp_log_size);
            pdev->dpc_rp_log_size = 0;
        }
    }
}
```

### 启用 DPC

**函数**: `dpc_enable()` (行 438-455)

```c
static void dpc_enable(struct pcie_device *dev)
{
    struct pci_dev *pdev = dev->port;
    int dpc = pdev->dpc_cap;
    u16 ctl;

    // 1. 清除 DPC Interrupt Status
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_STATUS,
                      PCI_EXP_DPC_STATUS_INTERRUPT);

    // 2. 读取当前控制
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_CTL, &ctl);

    // 3. 启用 DPC
    ctl &= ~PCI_EXP_DPC_CTL_EN_MASK;
    ctl |= PCI_EXP_DPC_CTL_EN_FATAL | PCI_EXP_DPC_CTL_INT_EN;
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_CTL, ctl);
}
```

### 禁用 DPC

**函数**: `dpc_disable()` (行 457-467)

```c
static void dpc_disable(struct pcie_device *dev)
{
    struct pci_dev *pdev = dev->port;
    int dpc = pdev->dpc_cap;
    u16 ctl;

    // 1. 禁用 DPC 触发和中断
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_CTL, &ctl);
    ctl &= ~(PCI_EXP_DPC_CTL_EN_FATAL | PCI_EXP_DPC_CTL_INT_EN);
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_CTL, ctl);
}
```

### DPC 中断处理

**函数**: `dpc_irq()` (行 370-398)

```c
static irqreturn_t dpc_irq(int irq, void *context)
{
    struct pcie_device *dev = context;
    struct pci_dev *pdev = dev->port;
    int dpc = pdev->dpc_cap;
    u16 status;

    // 1. 读取 DPC Status
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_STATUS, &status);

    // 2. 检查是否有 DPC 事件
    if (!(status & PCI_EXP_DPC_STATUS_INTERRUPT) || !PCI_POSSIBLE_ERROR(status))
        return IRQ_NONE;

    // 3. 清除中断状态
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_STATUS,
                      PCI_EXP_DPC_STATUS_INTERRUPT);

    // 4. 检查是否是 Trigger 事件
    if (status & PCI_EXP_DPC_STATUS_TRIGGER)
        return IRQ_WAKE_THREAD;

    return IRQ_HANDLED;
}
```

### DPC 错误处理

**函数**: `dpc_process_error()` (行 498-623)

```c
void dpc_process_error(struct pci_dev *pdev)
{
    int dpc = pdev->dpc_cap;
    u16 cap, ctl, status, reason, ext;
    struct pci_dev *dev, *temp_dev;
    pci_channel_state_t state;
    bool rp;
    bool trigger;

    // 1. 读取 DPC Capability
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_CAP, &cap);
    rp = (cap & PCI_EXP_DPC_CAP_RP_EXT) != 0;

    // 2. 读取 DPC Control
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_CTL, &ctl);

    // 3. 读取 DPC Status
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_STATUS, &status);

    // 4. 判断事件类型
    trigger = (status & PCI_EXP_DPC_STATUS_TRIGGER) != 0;

    // 5. 读取 DPC Reason
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_RP, &reason);

    // 6. 读取 Error Source Identifier
    pci_read_config_dword(pdev, dpc + PCI_EXP_DPC_RP_EXT, &ext);

    // 7. 查找错误源设备
    dev = pci_get_dpc_dev(pdev, ext, rp);
    if (!dev)
        return;

    // 8. 确定错误状态
    state = (trigger && (reason & PCI_EXP_DPC_RP_PIO_FEP)) ?
             pci_channel_io_frozen : pci_channel_io_normal;

    // 9. 执行恢复
    pcie_do_recovery(dev, state, dpc_reset_link);

    // 10. 清除 DPC 状态
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_STATUS, status);
}
```

### DPC 链路重置

**函数**: `dpc_reset_link()` (行 625-662)

```c
pci_ers_result_t dpc_reset_link(struct pci_dev *pdev)
{
    int dpc = pdev->dpc_cap;
    u16 cap, ctl, status;
    pci_ers_result_t ret;

    // 1. 读取 DPC Capability
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_CAP, &cap);

    // 2. 读取 DPC Control
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_CTL, &ctl);

    // 3. 读取 DPC Status
    pci_read_config_word(pdev, dpc + PCI_EXP_DPC_STATUS, &status);

    // 4. 检查 DPC 是否已触发
    if (!(status & PCI_EXP_DPC_STATUS_TRIGGER))
        return PCI_ERS_RESULT_NONE;

    // 5. 清除 DPC 状态
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_STATUS,
                      PCI_EXP_DPC_STATUS_TRIGGER);

    // 6. 禁用 DPC
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_CTL,
                      ctl & ~PCI_EXP_DPC_CTL_EN_FATAL);

    // 7. 重置链路
    ret = pci_reset_bridge(pdev, PCI_RESET_BRIDGE_FORCE);
    if (ret != PCI_ERS_RESULT_RECOVERED) {
        pci_err(pdev, "Failed to reset bridge\n");
        return ret;
    }

    // 8. 重新启用 DPC
    pci_write_config_word(pdev, dpc + PCI_EXP_DPC_CTL,
                      ctl | PCI_EXP_DPC_CTL_EN_FATAL);

    return PCI_ERS_RESULT_RECOVERED;
}
```

## Root Port 处理

Root Port 作为 DPC 的主要实施者：

```c
// 检查是否是 Root Port
if (pci_pcie_type(pdev) == PCI_EXP_TYPE_ROOT_PORT) {
    // Root Port 实现 DPC
    
    if (pdev->dpc_cap) {
        pci_info(pdev, "DPC capability enabled\n");
        
        // 读取 RP Extensions
        u16 cap;
        pci_read_config_word(pdev, pdev->dpc_cap + PCI_EXP_DPC_CAP, &cap);
        
        if (cap & PCI_EXP_DPC_CAP_RP_EXT) {
            pci_info(pdev, "DPC RP extensions supported\n");
            pci_info(pdev, "RP PIO log size: %u\n",
                     pdev->dpc_rp_log_size);
        }
    }
}
```

## 设备驱动使用

### 1. 检查 DPC 支持
```c
if (pdev->dpc_cap) {
    // 设备支持 DPC
    
    if (pdev->dpc_rp_extensions) {
        // RP Extensions 支持
        pci_info(&pdev->dev, "DPC RP extensions supported\n");
    }
}
```

### 2. DPC 恢复处理
```c
// 注册错误处理回调
static pci_ers_result_t my_dpc_reset(struct pci_dev *pdev)
{
    // 执行设备特定的恢复
    my_device_reset(pdev);
    
    return PCI_ERS_RESULT_RECOVERED;
}

// 注册回调
pci_set_drvdata(pdev, my_drvdata);
```

### 3. 监控 DPC 事件
```c
// DPC 事件通过 PCIe 服务处理
// 驱动不需要直接处理 DPC 中断
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_dpc_init()
                 ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_DPC)
                 ├─> pci_read_config_word()  // 读取 DPC Capability
                 └─> 配置 dpc_rp_log_size

DPC 事件:
  └─> dpc_irq()
       ├─> pci_read_config_word()  // 读取 DPC Status
       ├─> dpc_process_error()
       │    ├─> pci_get_dpc_dev()  // 查找错误源
       │    └─> pcie_do_recovery()  // 执行恢复
       └─> dpc_reset_link()
            ├─> pci_reset_bridge()  // 重置链路
            └─> pci_write_config_word()  // 重新启用 DPC
```

## 配置空间访问

### 查找 DPC Capability
```c
int dpc_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_DPC);
```

### 读取 DPC Capability
```c
u16 cap;
pci_read_config_word(dev, dpc_pos + PCI_EXP_DPC_CAP, &cap);

// 检查能力
bool rp_extensions = (cap & PCI_EXP_DPC_CAP_RP_EXT) != 0;
bool trigger_enable = (cap & PCI_EXP_DPC_CAP_TRIGGER) != 0;
bool completion_enable = (cap & PCI_EXP_DPC_CAP_COMPLETION) != 0;
```

### 读取/写入 DPC Control
```c
u16 ctl;

// 读取控制
pci_read_config_word(dev, dpc_pos + PCI_EXP_DPC_CTL, &ctl);

// 启用 DPC
ctl |= PCI_EXP_DPC_CTL_EN_FATAL;
ctl |= PCI_EXP_DPC_CTL_INT_EN;

// 写入控制
pci_write_config_word(dev, dpc_pos + PCI_EXP_DPC_CTL, ctl);
```

### 读取 DPC Status
```c
u16 status;
pci_read_config_word(dev, dpc_pos + PCI_EXP_DPC_STATUS, &status);

// 检查事件
bool trigger = (status & PCI_EXP_DPC_STATUS_TRIGGER) != 0;
bool completion = (status & PCI_EXP_DPC_STATUS_COMPLETION) != 0;
bool interrupt = (status & PCI_EXP_DPC_STATUS_INTERRUPT) != 0;
```

### 读取 DPC Reason
```c
u16 reason;
pci_read_config_word(dev, dpc_pos + PCI_EXP_DPC_RP, &reason);

// 获取触发原因
u16 trigger_reason = reason & PCI_EXP_DPC_RP_TRIGGER_REASON;

// 获取完成原因
u16 completion_reason = reason & PCI_EXP_DPC_RP_COMPLETION_REASON;
```

## DPC 错误隔离流程

```
错误发生
    ↓
DPC 检测错误
    ↓
DPC Trigger 触发
    ↓
隔离错误设备
    ↓
通知 PCIe 服务
    ↓
执行错误恢复
    ↓
DPC Completion
    ↓
重新启用链路
```

## 性能考虑

1. **错误隔离**: 防止错误传播到其他设备
2. **恢复时间**: DPC 恢复可能需要较长时间
3. **中断处理**: DPC 中断用于异步处理
4. **RP PIO Log**: 记录错误信息用于调试

## 调试信息

```c
// DPC 初始化
pci_info(pdev, "DPC capability enabled\n");

// RP Extensions
if (pdev->dpc_rp_extensions) {
    pci_info(pdev, "DPC RP extensions supported\n");
    pci_info(pdev, "RP PIO log size: %u\n",
             pdev->dpc_rp_log_size);
}

// DPC 事件
pci_err(pdev, "DPC error detected\n");
```

## 错误处理

1. **DPC 不支持**:
   ```c
   if (!pdev->dpc_cap)
       return;
   ```

2. **恢复失败**:
   ```c
   ret = dpc_reset_link(pdev);
   if (ret != PCI_ERS_RESULT_RECOVERED) {
       pci_err(pdev, "Failed to reset link\n");
       return ret;
   }
   ```

3. **RP PIO Log 无效**:
   ```c
   if (pdev->dpc_rp_log_size < PCIE_STD_NUM_TLP_HEADERLOG ||
       pdev->dpc_rp_log_size > PCIE_STD_MAX_TLP_HEADERLOG + 1) {
       pci_err(pdev, "RP PIO log size %u is invalid\n",
                pdev->dpc_rp_log_size);
       pdev->dpc_rp_log_size = 0;
   }
   ```

## 相关配置选项

- `CONFIG_PCIE_DPC`: 启用 DPC 支持
- `CONFIG_PCIEPORTBUS`: PCIe Port Bus 支持

## 参考资料

- PCIe Downstream Port Containment Specification
- PCIe Base Specification Revision 5.0+, Section 6.7
- Linux 内核源码: `drivers/pci/pcie/dpc.c`
- 头头文件: `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
