# Precision Time Measurement (PTM) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_PTM (0x1F)  
**规范参考**: PCIe PTM Specification  
**功能**: 提供精确的时间测量和同步

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x1F)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    4    PTM Capability
              - Bits [8:0]: Local Clock Granularity
              - Bit 9: PTM Requester Capable
              - Bit 10: PTM Responder Capable
              - Bit 11: Root Selected
0x08    4    PTM Control
              - Bit 0: PTM Enable
              - Bit 1: Root Selected
              - Bits [8:0]: Effective Granularity
```

### Granularity 值
| 值 | 粒度 | 描述 |
|-----|------|------|
| 0 | 未知 | 设备特定 |
| 1-254 | 纳秒 | 值 * 1 ns |
| 255 | >254ns | 大于 254 ns |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/pcie/ptm.c`  
**函数**: `pci_ptm_init()` (行 41-92)

```c
void pci_ptm_init(struct pci_dev *dev)
{
    u16 ptm;
    u32 cap;
    struct pci_dev *ups;

    // 1. 检查是否是 PCIe 设备
    if (!pci_is_pcie(dev))
        return;

    // 2. 查找 PTM capability
    ptm = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_PTM);
    if (!ptm)
        return;

    dev->ptm_cap = ptm;

    // 3. 分配保存缓冲区
    pci_add_ext_cap_save_buffer(dev, PCI_EXT_CAP_ID_PTM, sizeof(u32));

    // 4. 读取 PTM Capability
    pci_read_config_dword(dev, ptm + PCI_PTM_CAP, &cap);
    dev->ptm_granularity = FIELD_GET(PCI_PTM_GRANULARITY_MASK, cap);

    // 5. 查找上游 PTM 设备
    ups = pci_upstream_ptm(dev);
    if (ups) {
        // 6. 使用上游的粒度
        if (ups->ptm_granularity == 0)
            dev->ptm_granularity = 0;
        else if (ups->ptm_granularity > dev->ptm_granularity)
            dev->ptm_granularity = ups->ptm_granularity;
    } else if (cap & PCI_PTM_CAP_ROOT) {
        // 7. 设备是 PTM Root
        dev->ptm_root = 1;
    } else if (pci_pcie_type(dev) == PCI_EXP_TYPE_RC_END) {
        // 8. RC Endpoint 使用本地时钟粒度
        dev->ptm_granularity = 0;
    }

    // 9. 设置能力标志
    if (cap & PCI_PTM_CAP_RES)
        dev->ptm_responder = 1;
    if (cap & PCI_PTM_CAP_REQ)
        dev->ptm_requester = 1;

    // 10. Root Port 和 Upstream Port 自动启用
    if (pci_pcie_type(dev) == PCI_EXP_TYPE_ROOT_PORT ||
        pci_pcie_type(dev) == PCI_EXP_TYPE_UPSTREAM)
        pci_enable_ptm(dev, NULL);
}
```

### 启用 PTM

**函数**: `pci_enable_ptm()` (行 193-224)

```c
int pci_enable_ptm(struct pci_dev *dev, u8 *granularity)
{
    int rc;
    char clock_desc[8];

    // 1. 内部启用
    rc = __pci_enable_ptm(dev);
    if (rc)
        return rc;

    dev->ptm_enabled = 1;

    // 2. 返回粒度
    if (granularity)
        *granularity = dev->ptm_granularity;

    // 3. 格式化粒度描述
    switch (dev->ptm_granularity) {
    case 0:
        snprintf(clock_desc, sizeof(clock_desc), "unknown");
        break;
    case 255:
        snprintf(clock_desc, sizeof(clock_desc), ">254ns");
        break;
    default:
        snprintf(clock_desc, sizeof(clock_desc), "%uns",
                 dev->ptm_granularity);
        break;
    }

    pci_info(dev, "PTM enabled%s, %s granularity\n",
             dev->ptm_root ? " (root)" : "", clock_desc);

    return 0;
}
```

### 内部启用 PTM

**函数**: `__pci_enable_ptm()` (行 129-180)

```c
static int __pci_enable_ptm(struct pci_dev *dev)
{
    u16 ptm = dev->ptm_cap;
    struct pci_dev *ups;
    u32 ctrl;

    if (!ptm)
        return -EINVAL;

    // 1. 检查上游 PTM
    if (!dev->ptm_root) {
        ups = pci_upstream_ptm(dev);
    if (!ups || !ups->ptm_enabled)
            return -EINVAL;
    }

    // 2. 检查设备类型
    switch (pci_pcie_type(dev)) {
    case PCI_EXP_TYPE_ROOT_PORT:
        if (!dev->ptm_root)
            return -EINVAL;
        break;
    case PCI_EXP_TYPE_UPSTREAM:
        if (!dev->ptm_responder)
            return -EINVAL;
        break;
    case PCI_EXP_TYPE_ENDPOINT:
    case PCI_EXP_TYPE_LEG_END:
        if (!dev->ptm_requester)
            return -EINVAL;
        break;
    default:
        return -EINVAL;
    }

    // 3. 读取当前控制
    pci_read_config_dword(dev, ptm + PCI_PTM_CTRL, &ctrl);

    // 4. 启用 PTM
    ctrl |= PCI_PTM_CTRL_ENABLE;
    ctrl &= ~PCI_PTM_GRANULARITY_MASK;
    ctrl |= FIELD_PREP(PCI_PTM_GRANULARITY_MASK, dev->ptm_granularity);

    // 5. 设置 Root 标志
    if (dev->ptm_root)
        ctrl |= PCI_PTM_CTRL_ROOT;

    // 6. 写入控制
    pci_write_config_dword(dev, ptm + PCI_PTM_CTRL, ctrl);
    return 0;
}
```

### 禁用 PTM

**函数**: `pci_disable_ptm()` (行 245-252)

```c
void pci_disable_ptm(struct pci_dev *dev)
{
    if (dev->ptm_enabled) {
        __pci_disable_ptm(dev);
        dev->ptm_enabled = 0;
    }
}
```

### 内部禁用 PTM

**函数**: `__pci_disable_ptm()` (行 226-237)

```c
static void __pci_disable_ptm(struct pci_dev *dev)
{
    u16 ptm = dev->ptm_cap;
    u32 ctrl;

    if (!ptm)
        return;

    // 1. 读取当前控制
    pci_read_config_dword(dev, ptm + PCI_PTM_CTRL, &ctrl);

    // 2. 禁用 PTM
    ctrl &= ~(PCI_PTM_CTRL_ENABLE | PCI_PTM_CTRL_ROOT);

    // 3. 写入控制
    pci_write_config_dword(dev, ptm + PCI_PTM_CTRL, ctrl);
}
```

### 暂停/恢复 PTM

```c
void pci_suspend_ptm(struct pci_dev *dev)
{
    // 暂停时禁用 PTM，但保持启用标志
    if (dev->ptm_enabled)
        __pci_disable_ptm(dev);
}

void pci_resume_ptm(struct pci_dev *dev)
{
    // 恢复时重新启用 PTM
    if (dev->ptm_enabled)
        __pci_enable_ptm(dev);
}
```

## Root Port 处理

Root Port 作为 PTM Root，负责提供时间参考：

```c
// 检查是否是 Root Port
if (pci_pcie_type(dev) == PCI_EXP_TYPE_ROOT_PORT) {
    // Root Port 自动启用 PTM
    pci_enable_ptm(dev, NULL);
    
    // Root Port 的粒度影响所有下游设备
    dev_info(&dev->dev, "Root Port PTM enabled, granularity: %d\n",
             dev->ptm_granularity);
}
```

## 设备驱动使用

### 1. 检查 PTM 支持
```c
if (pdev->ptm_cap) {
    // 设备支持 PTM
    
    if (pdev->ptm_requester)
        dev_info(&pdev->dev, "PTM Requester capable\n");
    
    if (pdev->ptm_responder)
        dev_info(&pdev->dev, "PTM Responder capable\n");
}
```

### 2. 启用 PTM
```c
// 启用 PTM
u8 granularity;
int ret = pci_enable_ptm(pdev, &granularity);
if (ret) {
    dev_err(&pdev->dev, "Failed to enable PTM: %d\n", ret);
    return ret;
}

dev_info(&pdev->dev, "PTM enabled, granularity: %d ns\n", granularity);
```

### 3. 禁用 PTM
```c
// 禁用 PTM
pci_disable_ptm(pdev);
```

### 4. 检查 PTM 状态
```c
// 检查是否启用
if (pcie_ptm_enabled(pdev)) {
    dev_info(&pdev->dev, "PTM is enabled\n");
}

// 检查粒度
if (pdev->ptm_cap) {
    dev_info(&pdev->dev, "PTM granularity: %d ns\n",
             pdev->ptm_granularity);
}
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └`> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_ptm_init()
                 ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_PTM)
                 ├─> pci_read_config_dword()  // 读取 PTM Capability
                 ├─> pci_upstream_ptm()  // 查找上游 PTM
                 ├─> pci_add_ext_cap_save_buffer()  // 分配保存缓冲区
                 └─> pci_enable_ptm()  // Root Port 自动启用

驱动调用:
  └─> pci_enable_ptm()
       └─> __pci_enable_ptm()
            ├─> pci_upstream_ptm()  // 检查上游 PTM
            ├─> pci_read_config_dword()  // 读取控制
            └─> pci_write_config_dword()  // 启用 PTM
```

## 配置空间访问

### 查找 PTM Capability
```c
int ptm_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_PTM);
```

### 读取 PTM Capability
```c
u32 cap;
pci_read_config_dword(dev, ptm_pos + PCI_PTM_CAP, &cap);

// 获取粒度
u8 granularity = FIELD_GET(PCI_PTM_GRANULARITY_MASK, cap);

// 检查能力
bool requester_capable = (cap & PCI_PTM_CAP_REQ) != 0;
bool responder_capable = (cap & PCI_PTM_CAP_RES) != 0;
bool root_capable = (cap & PCI_PTM_CAP_ROOT) != 0;
```

### 读取/写入 PTM Control
```c
u32 ctrl;

// 读取控制
pci_read_config_dword(dev, ptm_pos + PCI_PTM_CTRL, &ctrl);

// 检查是否启用
bool enabled = (ctrl & PCI_PTM_CTRL_ENABLE) != 0;

// 获取有效粒度
u8 effective_granularity = FIELD_GET(PCI_PTM_GRANULARITY_MASK, ctrl);

// 启用 PTM
ctrl |= PCI_PTM_CTRL_ENABLE;
ctrl &= ~PCI_PTM_GRANULARITY_MASK;
ctrl |= FIELD_PREP(PCI_PTM_GRANULARITY_MASK, granularity);

// 写入控制
pci_write_config_dword(dev, ptm_pos +` PCI_PTM_CTRL, ctrl);
```

## PTM 消息流程

```
PTM Requester 发送 PTM Request
    ↓
PTM Responder 处理请求
    ↓
PTM Responder 发送 PTM Response
    ↓
PTM Requester 接收响应
    ↓
计算时间差
```

## 粒度计算

```
Effective Granularity = max(Local Granularity, Upstream Granularity)

如果 Upstream Granularity = 0:
    Effective Granularity = Local Granularity

如果 Local Granularity = 0:
    Effective Granularity = Upstream Granularity
```

## 性能考虑

1. **粒度**: 较小的粒度提供更高的精度
2. **延迟**: PTM 消息会增加延迟
3. **开销**: PTM 需要额外的消息处理
4. **同步**: PTM Root 需要稳定的时钟源

## 调试信息

```c
// PTM 启用信息
pci_info(dev, "PTM enabled%s, %s granularity\n",
         dev->ptm_root ? " (root)" : "", clock_desc);

// 粒度信息
dev_info(&pdev->dev, "PTM granularity: %d ns\n",
         pdev->ptm_granularity);
```

## 错误处理

1. **PTM 不支持**:
   ```c
   if (!dev->ptm_cap)
       return -EINVAL;
   ```

2. **上游 PTM 未启用**:
   ```c
   ups = pci_upstream_ptm(dev);
   if (!ups || !ups->ptm_enabled)
       return -EINVAL;
   ```

3. **启用失败**:
   ```c
   ret = pci_enable_ptm(pdev, &granularity);
   if (ret) {
       dev_err(&pdev->dev, "Failed to enable PTM: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCIE_PTM`: 启用 PTM 支持
- `CONFIG_DEBUG_FS`: PTM debugfs 支持

## 参考资料
- PCIe Precision Time Measurement Specification
- PCIe Base Specification Revision 5.0+, Section 7.9
- Linux 内核源码: `drivers/pci/pcie/ptm.c`
- 头文件: `include/linux/ptm.h`, `include/uapi/linux/pci_regs.h`
