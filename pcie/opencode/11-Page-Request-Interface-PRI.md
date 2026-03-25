# Page Request Interface (PRI) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_PRI (0x13)  
**规范参考**: PCIe PRI Specification  
**功能**: 支持设备向 IOMMU 请求页面

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x13)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    2    PRI Capability
              - Bit 0: Stop
              - Bit 1: PASID Required
0x06    2    PRI Control
              - Bit 0: PRI Enable
              - Bit 1: Reset
0x08    4    PRI Status
              - Bit 0: Stopped
              - Bits [3:2]: PF Count
              - Bits [11:4]: Outstanding Requests
0x0C    4    PRI Capacity
              - Bits [13:0]: Page Request Capacity
              - Bits [31:19]: Reserved
0x10    4    PRI Allocation Request
0x14    4    PRI Outstanding Page Data
```

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/ats.c`  
**函数**: `pci_pri_init()` (行 209-221)

```c
void pci_pri_init(struct pci_dev *pdev)
{
    u16 status;

    // 1. 查找 PRI capability
    pdev->pri_cap = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_PRI);

    if (!pdev->pri_cap)
        return;

    // 2. 读取 PRI Status
    pci_read_config_word(pdev, pdev->pri_cap + PCI_PRI_STATUS, &status);

    // 3. 检查是否需要 PASID
    if (status & PCI_PRI_STATUS_PASID)
        pdev->pasid_required = 1;
}
```

### 启用 PRI

**函数**: `pci_enable_pri()` (行 230-268)

```c
int pci_enable_pri(struct pci_dev *pdev, u32 reqs)
{
    u16 control, status;
    u32 max_requests;
    int pri = pdev->pri_cap;

    // 1. VF 不实现 PRI
    if (pdev->is_virtfn) {
        if (pci_physfn(pdev)->pri_enabled)
            return 0;
        return -EINVAL;
    }

    // 2. 检查是否已启用
    if (WARN_ON(pdev->pri_enabled))
        return -EBUSY;

    // 3. 检查 PRI capability
    if (!pri)
        return -EINVAL;

    // 4. 检查 PRI 是否已停止
    pci_read_config_word(pdev, pri + PCI_PRI_STATUS, &status);
    if (!(status & PCI_PRI_STATUS_STOPPED))
        return -EBUSY;

    // 5. 读取最大请求数量
    pci_read_config_dword(pdev, pri + PCI_PRI_MAX_REQ, &max_requests);
    reqs = min(max_requests, reqs);
    pdev->pri_reqs_alloc = reqs;

    // 6. 写入分配请求数量
    pci_write_config_dword(pdev, pri + PCI_PRI_ALLOC_REQ, reqs);

    // 7. 启用 PRI
    control = PCI_PRI_CTRL_ENABLE;
    pci_write_config_word(pdev, pri + PCI_PRI_CTRL, control);

    pdev->pri_enabled = 1;

    return 0;
}
```

### 禁用 PRI

**函数**: `pci_disable_pri()` (行 276-296)

```c
void pci_disable_pri(struct pci_dev *pdev)
{
    u16 control;
    int pri = pdev->pri_cap;

    // 1. VF 不需要单独禁用
    if (pdev->is_virtfn)
        return;

    // 2. 检查是否已启用
    if (WARN_ON(!pdev->pri_enabled))
        return;

    // 3. 检查 PRI capability
    if (!pri)
        return;

    // 4. 禁用 PRI
    pci_read_config_word(pdev, pri + PCI_PRI_CTRL, &control);
    control &= ~PCI_PRI_CTRL_ENABLE;
    pci_write_config_word(pdev, pri + PCI_PRI_CTRL, control);

    pdev->pri_enabled = 0;
}
```

### 重置 PRI

**函数**: `pci_reset_pri()` (行 329-347)

```c
int pci_reset_pri(struct pci_dev *pdev)
{
    u16 control;
    int pri = pdev->pri_cap;

    // 1. VF 不需要单独重置
    if (pdev->is_virtfn)
        return 0;

    // 2. 检查是否已启用
    if (WARN_ON(pdev->pri_enabled))
        return -EBUSY;

    // 3. 检查 PRI capability
    if (!pri)
        return -EINVAL;

    // 4. 发送重置命令
    control = PCI_PRI_CTRL_RESET;
    pci_write_config_word(pdev, pri + PCI_PRI_CTRL, control);

    return 0;
}
```

### 恢复 PRI 状态

**函数**: `pci_restore_pri_state()` (行 303-320)

```c
void pci_restore_pri_state(struct pci_dev *pdev)
{
    u16 control = PCI_PRI_CTRL_ENABLE;
    u32 reqs = pdev->pri_reqs_alloc;
    int pri = pdev->pri_cap;

    // 1. VF 不需要单独恢复
    if (pdev->is_virtfn)
        return;

    // 2. 检查是否已启用
    if (!pdev->pri_enabled)
        return;

    // 3. 检查 PRI capability
    if (!pri)
        return;

    // 4. 写入分配请求数量
    pci_write_config_dword(pdev, pri + PCI_PRI_ALLOC_REQ, reqs);

    // 5. 启用 PRI
    pci_write_config_word(pdev, pri + PCI_PRI_CTRL, control);
}
```

### 检查 PRI 支持

**函数**: `pci_pri_supported()` (行 370-377)

```c
bool pci_pri_supported(struct pci_dev *pdev)
{
    // VF 共享 PF 的 PRI
    if (pci_physfn(pdev)->pri_cap)
        return true;

    return false;
}
```

### 检查 PRG Response PASID Required

**函数**: `pci_prg_resp_pasid_required()` (行 356-362)

```c
int pci_prg_resp_pasid_required(struct pci_dev *pdev)
{
    // VF 返回 PF 的状态
    if (pdev->is_virtfn)
        pdev = pci_physfn(pdev);

    return pdev->pasid_required;
}
```

## Root Port 处理

Root Port 通常不实现 PRI capability，因为它是 Bridge 设备。PRI 主要用于 Endpoint 设备与 IOMMU 交互。

## 设备驱动使用

### 1. 检查 PRI 支持
```c
if (pci_pri_supported(pdev)) {
    // 设备支持 PRI
    
    // 检查是否需要 PASID
    if (pdev->pasid_required) {
        dev_info(&pdev->dev, "PASID required for PRI\n");
    }
}
```

### 2. 启用 PRI
```c
// 分配页面请求数量
u32 reqs = 32;

// 启用 PRI
int ret = pci_enable_pri(pdev, reqs);
if (ret) {
    dev_err(&pdev->dev, "Failed to enable PRI: %d\n", ret);
    return ret;
}

dev_info(&pdev->dev, "PRI enabled, requests: %u\n", reqs);
```

### 3. 禁用 PRI
```c
// 禁用 PRI
pci_disable_pri(pdev);
```

### 4. 重置 PRI
```c
// 重置 PRI
int ret = pci_reset_pri(pdev);
if (ret) {
    dev_err(&pdev->dev, "Failed to reset PRI: %d\n", ret);
    return ret;
}
```

## 调用流程

```
IOMMU 驱动:
  └─> pci_enable_pri(pdev, reqs)
       ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_PRI)
       ├─> pci_read_config_word()  // 读取 PRI Status
       ├─> pci_read_config_dword()  // 读取 Max Requests
       ├─> pci_write_config_dword()  // 写入 Alloc Requests
       └─> pci_write_config_word()  // 启用 PRI

设备挂起:
  └─> pci_init_capabilities()
       └─> pci_pri_init()
            ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_PRI)
            └─> pci_read_config_word()  // 读取 PRI Status
            └─> 检查 PASID Required
```

## 配置空间访问

### 查找 PRI Capability
```c
int pri_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_PRI);
```

### 读取 PRI Capability
```c
u16 cap;
pci_read_config_word(dev, pri_pos + PCI_PRI_CAP, &cap);

// 检查能力
u32 pr_cap = cap & PCI_PRI_CAP_PAGE_REQ_CAP;

// 检查 PASID Required
bool pasid_required = (cap & PCI_PRI_CAP_PASID) != 0;
```

### 读取 PRI Status
```c
u16 status;
pci_read_config_word(dev, pri_pos + PCI_PRI_STATUS, &status);

// 检查是否停止
bool stopped = (status & PCI_PRI_STATUS_STOPPED) != 0;

// 获取 PF Count
u8 pf_count = FIELD_GET(PCI_PRI_STATUS_PF_COUNT, status);

// 获取 Outstanding Requests
u16 outstanding = FIELD_GET(PCI_PRI_STATUS_OUTSTANDING, status);
```

### 读取/写入 PRI Control
```c
u16 ctrl;

// 读取控制
pci_read_config_word(dev, pri_pos + PCI_PRI_CTRL, &ctrl);

// 检查是否启用
bool enabled = (ctrl & PCI_PRI_CTRL_ENABLE) != 0;

// 启用 PRI
ctrl |= PCI_PRI_CTRL_ENABLE;

// 重置 PRI
ctrl |= PCI_PRI_CTRL_RESET;

// 写入控制
pci_write_config_word(dev, pri_pos + PCI_PRI_CTRL, ctrl);
```

### 读取/写入 Max Requests
```c
u32 max_reqs;

// 读取最大请求数量
pci_read_config_dword(dev, pri_pos + PCI_PRI_MAX_REQ, &max_reqs);

// 写入分配请求数量
pci_write_config_dword(dev, pri_pos + PRI_PRI_ALLOC_REQ, reqs);
```

## PRI 与 ATS/PASID 的关系

```
ATS → PRI → PASID

ATS: 地址转换服务
PRI: 页面请求接口
PASID: 进程地址空间 ID

通常一起使用以支持 IOMMU 功能
```

## 性能考虑

1. **请求数量**: 影响 IOMMU 的页面请求吞吐量
2. **VF 共享**: VF 共享 PF 的 PRI 配置
3. **PASID 要求**: 某些设备要求 PASID 才能使用 PRI
4. **停止状态**: PRI 可以停止接受新请求

## 调试信息

```c
// PRI 启用
dev_info(&pdev->dev, "PRI enabled, requests: %u\n", reqs);

// PASID 要求
if (pdev->pasid_required) {
    dev_info(&pdev->dev, "PASID required for PRI\n");
}
```

## 错误处理

1. **PRI 不支持**:
   ```c
   if (!pdev->pri_cap)
       return -EINVAL;
   ```

2. **PRI 已停止**:
   ```c
   pci_read_config_word(pdev, pri + PCI_PRI_STATUS, &status);
   if (!(status & PCI_PRI_STATUS_STOPPED))
       return -EBUSY;
   ```

3. **启用失败**:
   ```c
   ret = pci_enable_pri(pdev, reqs);
   if (ret) {
       dev_err(&pdev->dev, "Failed to enable PRI: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCI_PRI`: 启用 PRI 支持
- `CONFIG_PCI_ATS`: ATS 支持（PRI 的依赖）
- `CONFIG_PCI_PASID`: PASID 支持（PRI 的依赖）

## 参考资料

- PCIe Page Request Interface Specification
- PCIe Base Specification Revision 5.0+, Section 10.6
- Linux 内核源码: `drivers/pci/ats.c`
- 头文件: `include/linux/pci-ats.h`, `include/uapi/linux/pci_regs.h`
