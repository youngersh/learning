# Process Address Space ID (PASID) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_PASID (0x1B)  
**规范参考**: PCIe PASID Specification  
**功能**: 支持进程地址空间标识，用于 IOMMU 和 SR-IOV

## PCIe SPEc 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x1B)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    2    PASID Capability
              - Bits [4:0]: Exec Support
              - Bit 5: Priv Support
0x06    2    PASID Control
              - Bit 0: PASID Enable
              - Bits [4:1]: PASID Exec Enable
              - Bit 5: PASID Priv Enable
```

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/ats.c`  
**函数**: `pci_pasid_init()` (行 381-384)

```c
void pci_pasid_init(struct pci_dev *pdev)
{
    // 1. 查找 PASID capability
    pdev->pasid_cap = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_PASID);
}
```

### 检查 PASID 支持

**函数**: `pci_pasid_supported()` (行 497-514)

```c
int pci_pasid_supported(struct pci_dev *pdev)
{
    u16 supported;
    int pasid;

    // 1. VF 返回 PF 的状态
    if (pdev->is_virtfn)
        pdev = pci_physfn(pdev);

    // 2. 检查 PASID capability
    pasid = pdev->pasid_cap;
    if (!pasid)
        return -EINVAL;

    // 3. 读取 PASID Capability
    pci_read_config_word(pdev, pasid + PCI_PASID_CAP, &supported);

    // 4. 返回支持的功能
    supported &= PCI_PASID_CAP_EXEC | PCI_PASID_CAP_PRIV;

    return supported;
}
```

### 获取最大 PASID 数量

**函数**: `pci_max_pasids()` (行 524-540)

```c
int pci_max_pasids(struct pci_dev *pdev)
{
    u16 supported;
    int pasid;
    u16 width;

    // 1. VF 返回 PF 的状态
    if (pdev->is_virtfn)
        pdev = pci_physfn(pdev);

    // 2. 检查 PASID capability
    pasid = pdev->pasid_cap;
    if (!pasid)
        return -EINVAL;

    // 3. 读取 PASID Capability
    pci_read_config_word(pdev, pasid + PCI_PASID_CAP, &supported);

    // 4. 获取 PASID Width
    width = FIELD_GET(PCI_PASID_CAP_WIDTH, supported);

    // 5. 返回最大 PASID 数量
    return (1 << width;
}
```

### 启用 PASID

**函数**: `pci_enable_pasid()` (行 395-437)

```c
int pci_enable_pasid(struct pci_dev *pdev, int features)
{
    u16 control, supported;
    int pasid;

    // 1. VF 不实现 PASID
    if (pdev->is_virtfn) {
        if (pci_physfn(pdev)->pasid_enabled)
            return 0;
        return -EINVAL;
    }

    // 2. 检查是否已启用
    if (WARN_ON(pdev->pasid_enabled))
        return -EBUSY;

    // 3. 检查 PASID capability
    pasid = pdev->pasid_cap;
    if (!pasid)
        return -EINVAL;

    // 4. 检查是否支持 ACS
    if (!pci_acs_path_enabled(pdev, NULL, PCI_ACS_RR | PCI_ACS_UF))
        return -EINVAL;

    // 5. 读取 PASID Capability
    pci_read_config_word(pdev, pasid + PCI_PASID_CAP, &supported);

    // 6. 检查请求的功能是否支持
    supported &= PCI_PASID_CAP_EXEC | PCI_PASID_CAP_PRIV;
    if ((supported & features) != features)
        return -EINVAL;

    // 7. 配置 PASID Control
    control = PCI_PASID_CTRL_ENABLE | features;
    pdev->pasid_features = features;

    // 8. 写入控制
    pci_write_config_word(pdev, pasid + PCI_PASID_CTRL, control);

    pdev->pasid_enabled = 1;

    return 0;
}
```

### 禁用 PASID

**函数**: `pci_disable_pasid()` (行 444-463)

```c
void pci_disable_pasid(struct pci_dev *pdev)
{
    u16 control;
    int pasid;

    // 1. VF 不需要单独禁用
    if (pdev->is_virtfn)
        return;

    // 2. 检查是否已启用
    if (WARN_ON(!pdev->pasid_enabled))
        return;

    // 3. 检查 PASID capability
    pasid = pdev->pasid_cap;
    if (!pasid)
        return;

    // 4. 禁用 PASID
    control = 0;
    pci_write_config_word(pdev, pasid + PCI_PASID_CTRL, control);

    pdev->pasid_enabled = 0;
}
```

### 恢复 PASID 状态

**函数**: `pci_restore_pasid_state()` (行 469-485)

```c
void pci_restore_pasid_state(struct pci_dev *pdev)
{
    u16 control;
    int pasid;

    // 1. VF 不需要单独恢复
    if (pdev->is_virtfn)
        return;

    // 2. 检查是否已启用
    if (!pdev->pasid_enabled)
        return;

    // 3. 检查 PASID capability
    pasid = pdev->pasid_cap;
    if (!pasid)
        return;

    // 4. 恢复 PASID Control
    control = PCI_PASID_CTRL_ENABLE | pdev->pasid_features;
    pci_write_config_word(pdev, pasid + PCI_PASID_CTRL, control);
}
```

### 获取 PASID 状态

**函数**: `pci_pasid_status()` (行 554-573)

```c
int pci_pasid_status(struct pci_dev *pdev)
{
    int pasid;
    u16 ctrl;

    // 1. VF 返回 PF 的状态
    if (pdev->is_virtfn)
        pdev = pci_physfn(pdev);

    // 2. 检 � 读取 PASID Control
    pasid = pdev->pasid_cap;
    if (!pasid)
        return -EINVAL;

    pci_read_config_word(pdev, pasid + PCI_PASID_CTRL, &ctrl);

    // 3. 返回控制状态
    ctrl &= PCI_PASID_CTRL_ENABLE | PCI_PASID_CTRL_EXEC | PCI_PASID_CTRL_PRIV;

    return ctrl;
}
```

## Root Port 处理

Root Port 通常不实现 PASID capability，因为它是 Bridge 设备。PASID 主要用于 Endpoint 设备与 IOMMU 交互。

## 设备驱动使用

### 1. 检查 PASID 支持
```c
int supported = pci_pasid_supported(pdev);
if (supported < 0) {
    dev_err(&pdev->dev, "PASID not supported\n");
    return;
}

// 检查支持的功能
if (supported & PCI_PASID_CAP_EXEC)
    dev_info(&pdev->dev, "PASID Exec supported\n");

if (supported & PCI_PASID_CAP_PRIV)
    dev_info(&pdev->dev, "PASID Priv supported\n");
```

### 2. 获取最大 PASID 数量
```c
int max_pasids = pci_max_pasids(pdev);
if (max_pasids < 0) {
    dev_err(&pdev->dev, "Failed to get max PASIDs: %d\n", max_pasids);
    return;
}

dev_info(&pdev->dev, "Max PASIDs: %d\n", max_pasids);
```

### 3. 启用 PASID
```c
// 选择要启用的功能
int features = 0;
if (pdev->pasid_cap & PCI_PASID_CAP_EXEC)
    features |= PCI_PASID_CAP_EXEC;

if (pdev->pasid_cap & PCI_PASID_CAP_PRIV)
    features |= PCI_PID_CAP_PRIV;

// 启用 PASID
int ret = pci_enable_pasid(pdev, features);
if (ret) {
    dev_err(&pdev->dev, "Failed to enable PASID: %d\n", ret);
    return ret;
}

dev_info(&pdev->dev, "PASID enabled\n");
```

### 4. 禁用 PASID
```c
// 禁用 PASID
pci_disable_pasid(pdev);
```

### 5. 检查 PASID 状态
```c
// 获取 PASID 状态
int status = pci_pasid_status(pdev);

if (status & PCI_PASID_CTRL_ENABLE)
    dev_info(&pdev->dev, "PASID enabled\n");

if (status & PCI_PASID_CTRL_EXEC)
    dev_info(&pdev->dev, "PASID Exec enabled\n");

if (status & PCI_PASID_CTRL_PRIV)
    dev_info(&pdev->dev, "PASID Priv enabled\n");
```

## 调用流程

```
IOMMU 驱动:
  └─> pci_enable_pasid(pdev, features)
       ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_PASID)
       ├─> pci_acs_path_enabled()  // 检查 ACS 支持
       ├─> pci_read_config_word()  // 读取 PASID Capability
       └─> pci_write_config_word()  // 启用 PASID

设备挂起:
  └─> pci_init_capabilities()
       └─> pci_pasid_init()
            └─> pci_find_ext_capability(PCI_EXT_CAP_ID_PASID)
```

## 配置空间访问

### 查找 PASID Capability
```c
int pasid_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_PASID);
```

### 读取 PASID Capability
```c
u16 cap;
pci_read_config_word(dev, pasid_pos + PCI_PASID_CAP, &cap);

// 检查能力
bool exec_support = (cap & PCI_PASID_CAP_EXEC) != 0;
bool priv_support = (cap & PCI_PASID_CAP_PRIV) != 0;

// 获取 PASID Width
u8 width = FIELD_GET(PCI_PASID_CAP_WIDTH, cap);

// 计算最大 PASID 数量
int max_pasids = 1 << width;
```

### 读取/写入 PASID Control
```c
u16 ctrl;

// 读取控制
pci_read_config_word(dev, pasid_pos + PCI_PASID_CTRL, &ctrl);

// 检查是否启用
bool enabled = (ctrl & PCI_PASID_CTRL_ENABLE) != 0;

// 检查 Exec 是否启用
bool exec_enabled = (ctrl & PCI_PASID_CTRL_EXEC) != 0;

// 检查 Priv 是否启用
bool priv_enabled = (ctrl & PCI_PASID_CTRL_PRIV) != 0;

// 启用 PASID
ctrl |= PCI_PASID_CTRL_ENABLE;
ctrl |= features;

// 写入控制
pci_write_config_word(dev, pasid_pos + PCI_PASID_CTRL, ctrl);
```

## PASID 与 ATS/PRI 的关系

```
ATS → PRI → PASID

ATS: 地址转换服务
PRI: 页面请求接口
PASID: 进程地址空间 ID

通常一起使用以支持 IOMMU 功能
```

## 性能考虑

1. **PASID 数量**: 影响 IOMMU 的地址空间管理
2. **Exec 权限**: 提供 execute 权限功能
3. **Priv 权限**: 提供 privileged 模式
4. **ACS 要求**: 启用 PASID 需要 ACS 支持

## 调试信息

```c
// PASID 启用
dev_info(&pdev->dev, "PASID enabled\n");

// 最大 PASID 数量
int max_pasids = pci_max_pasids(pdev);
dev_info(&pdev->dev, "Max PASIDs: %d\n", max_pasids);
```

## 错误处理

1. **PASID 不支持**:
   ```c
   if (!pdev->pasid_cap)
       return -EINVAL;
   ```

2. **ACS 不支持**:
   ```c
   if (!pci_acs_path_enabled(pdev, NULL, PCI_ACS_RR | PCI_ACS_UF))
       return -EINVAL;
   ```

3. **功能不支持**:
   ```c
   if ((supported & features) != features)
       return -EINVAL;
   ```

4. **启用失败**:
   ```c
   ret = pci_enable_pasid(pdev, features);
   if (ret) {
       dev_err(&pdev->dev, "Failed to enable PASID: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCI_PASID`: 启用 PASID 支持
- `CONFIG_PCI_ATS`: ATS 支持（PASID 的依赖）
- `CONFIG_PCI_PRI`: PRI 支持（PASID 的依赖）
- `CONFIG_PCI_ACS`: ACS 支持（PASID 的依赖）

## 参考资料

- PCIe Process Address Space ID Specification
- PCIe Base Specification Revision 5.0+, Section 10.7
- Linux 内核源码: `drivers/pci/ats.c`
- 头文件: `include/linux/pci-ats.h`, `include/uapi/linux/pci_regs.h`
