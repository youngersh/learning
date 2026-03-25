# Access Control Services (ACS) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_ACS (0x0D)  
**规范参考**: PCIe ACS Specification  
**功能**: 访问控制服务，用于 IOMMU 和 P2P 拓扑

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x0D)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    2    ACS Capability
              - Bit 0: Source Validation
              - Bit 1: Redirect Request
              - Bit 2: Translation Blocking
              - Bit 3: P2P Request Blocking
              - Bit 4: P2P Completion Blocking
              - Bit 5: Direct Translated P2P
0x06    2    ACS Control
              - Bit 0: ACS Enable
0x08    4    ACS Extended Control
              - Bit 0: Egress Translation Blocking
              - Bit 1: Direct Translated P2P Request Blocking
              - Bit 2: P2P Request Completion Blocking
              Bit 3: P2P Request Completion Blocking
```

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/pci.c`  
**函数**: `pci_acs_init()` (行 3645-3679)

```c
void pci_acs_init(struct pci_dev *dev)
{
    int pos;

    // 1. 查找 ACS capability
    pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ACS);
    if (!pos)
        return;

    // 2. 读取 ACS Capability
    pci_read_config_word(dev, pos + PCI_ACS_CAP, &caps);

    // 3. 启用 Source Validation
    if (caps & PCI_ACS_CAP_SV)
        pci_write_config_word(dev, pos + PCI_ACS_CTRL,
                      PCI_ACS_CTRL_SV);

    // 4. 启用 Redirect Request
    if (caps & PCI_ACS_CAP_RR)
        pci_write_config_word(dev, pos + PCI_ACS_CTRL,
                      PCI_ACS_CTRL_RR);

    // 5. 启用 Translation Blocking
    if (caps & PCI_ACS_CAP_TB)
        pci_write_config_word(dev, pos + PCI_ACS_CTRL,
                      PCI_ACS_CTRL_TB);

    // 6. 启用 P2P Request Blocking
    if (caps & PCI_ACS_CAP_PRP)
        pci_write_config_word(dev, pos + PCI_ACS_CTRL,
                      PCI_ACS_CTRL_PRP);

    // 7. 启用 P2P Completion Blocking
    if (caps & PCI_ACS_CAP_PCP)
        pci_write_config_word(dev, pos + PCI_ACS_CTRL,
                      PCI_ACS_CTRL_PCP);

    // 8. 启用 ACS Enable
    pci_write_config_word(dev, pos + PCI_ACS_CTRL, PCI_ACS_CTRL_EN);
}
```

### 检查 ACS 路径使能

**函数**: `pci_acs_path_enabled()` (行 389-405)

```c
bool pci_acs_path_enabled(struct pci_dev *dev, struct pci_dev *end,
                          u16 acs_flags)
{
    struct pci_dev *pdev;
    u16 acs_cap;
    u16 acs_ctrl, acs_ext;
    u16 acs_cap2, acs_ctrl2;

    // 1. 遍历从当前设备到 Root Port
    while (dev != end) {
        // 2. 检查当前设备是否是 PF
        if (dev->is_virtfn)
            pdev = pci_physfn(dev);
        else
            pdev = dev;

        // 3. 检查 ACS capability
        acs_cap = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_ACS);
        if (!acs_cap)
            return false;

        // 4. 读取 ACS Capability
        pci_read_config_word(pdev, acs_cap + PCI_ACS_CAP, &acs_cap);

        // 5. 检查是否支持请求的功能
        if ((acs_cap & acs_flags) != acs_flags)
            return false;

        // 6. 读取 ACS Control
        pci_read_config_word(pdev, acs_cap + PCI_ACS_CTRL, &acs_ctrl);
        if (!(acs_ctrl & PCI_ACS_CTRL_EN))
            return false;

        // 7. 检查扩展控制
        if (acs_flags & PCI_ACS_EGRESS_TL)
            pci_read_config_dword(pdev, acs_cap + PCI_ACS_EGRESS_CTRL,
                          &acs_ext);

        // 8. 检查 Egress Translation Blocking
        if (acs_flags & PCI_ACS_EGRESS_TL &&
            (acs_ext & PCI_ACS_EGRESS_TL_BLOCK))
            return false;

        // 9. 检查 P2P Blocking
        if (acs_flags & PCI_ACS_P2P_REQ &&
            (acs_ext & PCI_ACS_P2P_REQ_BLOCK))
            return false;

        // 10. 移动到上游设备
        dev = pci_upstream_bridge(dev);
        if (!dev)
            return false;
    }

    return true;
}
```

### 检查设备特定 ACS 支持

**函数**: `pci_dev_specific_acs_enabled()` (行 941-965)

```c
int pci_dev_specific_acs_enabled(struct pci_dev *dev, u16 acs_flags)
{
    // 由平台特定代码实现
    return -ENOTTY;
}
```

### 启用设备特定 ACS

**函数**: `pci_dev_specific_enable_acs()` (行 967-965)

```c
int pci_dev_specific_enable_acs(struct pci_dev *dev)
{
    // 由平台特定代码实现
    return -ENOTTY;
}
```

### 禁用设备特定 ACS 重定向

**函数**: `pci_dev_specific_disable_acs_redir()` (行 967-965)

```c
int pci_dev_specific_disable_acs_redir(struct pci_dev *dev)
{
    // 由平台特定代码实现
    return -ENOTTY;
}
```

## Root Port 处理

Root Port 作为 ACS 策略的起点，需要正确配置 ACS：

```c
// Root Port 自动启用 ACS
if (pci_pcie_type(dev) == PCI_EXP_TYPE_ROOT_PORT) {
    // Root Port 的 ACS 配置影响所有下游设备
    dev_info(&dev->dev, "Root Port ACS enabled\n");
}
```

## 设备驱动使用

### 1. 检查 ACS 支持
```c
// 检查 ACS 是否支持
int pos = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_ACS);
if (!pos) {
    dev_info(&pdev->dev, "ACS not supported\n");
    return;
}

// 读取 ACS Capability
u16 caps;
pci_read_config_word(pdev, pos + PCI_ACS_CAP, &caps);

dev_info(&pdev->dev, "ACS capabilities: %#04x\n", caps);
```

### 2. 检查 ACS 路径
```c
// 检查 Redirect Request 路径
if (pci_acs_path_enabled(pdev, NULL, PCI_ACS_CAP_RR)) {
    dev_info(&pdev->dev, "ACS Redirect Request enabled\n");
}

// 检查 Translation Blocking 路径
if (pci_acs_path_enabled(pdev, NULL, PCI_ACS_CAP_TB)) {
    dev_info(&pdev->dev, "ACS Translation Blocking enabled\n");
}

// 检查 P2P Request Blocking 路径
if (pci_acs_path_enabled(pdev, NULL, PCI_ACS_CAP_PRP)) {
    dev_info(&pdev->dev, "ACS P2P Request Blocking enabled\n");
}
```

### 3. 配置 ACS（设备特定）
```c
// 某些设备可能需要特殊的 ACS 配置
int ret = pci_dev_specific_enable_acs(pdev);
if (ret) {
    dev_info(&pdev->dev, "Device specific ACS enabled\n");
}
```

### 4. 禁用 Redirect（设备特定）
```c
// 某些设备可能需要禁用 Redirect
int ret = pci_dev_specific_disable_acs_redir(pdev);
if (ret) {
    dev_info(&pdev->dev, "Device specific ACS redirect disabled\n");
}
```

## 调用流程

```
设备挂起:
  └─> pci_init_capabilities()
       └─> pci_acs_init()
            ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_ACS)
            ├─> pci_read_config_word()  // 读取 ACS Capability
            ├─> pci_write_config_word()  // 启用 SV
            ├─> pci_write_config_word()  // 启用 RR
            ├─> pci_write_config_word()  // 启用 TB
            ├─> pci_write_config_word() // 启用 PRP
            └─> pci_write_config_word()  // 启用 PCP
            └─> pci_write_config_word()  // 启用 EN

IOMMU 驱动:
  └─> pci_acs_path_enabled()
       ├─> 遍历到 Root Port
       ├─> pci_find_ext_capability()  // 查找 ACS
       ├─> pci_read_config_word()  // 读取 ACS Capability
       └─> pci_read_config_word()  // 读取 ACS Control
       └─> 检查功能支持
       └─> 检查扩展控制
       └─> 检查阻塞状态
```

## 配置空间访问

### 查找 ACS Capability
```c
int acs_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ACS);
```

### 读取 ACS Capability
```c
u16 caps;
pci_read_config_word(dev, acs_pos + PCI_ACS_CAP, &caps);

// 检查能力
bool sv_support = (caps & PCI_ACS_CAP_SV) != 0;
bool rr_support = (caps & PCI_ACS_CAP_RR) != 0;
bool tb_support = (caps & PCI_ACS_CAP_TB) != 0;
bool prp_support = (caps & PCI_ACS_CAP_PRP) != 0;
bool pcp_support = (caps & PCI_ACS_CAP_PCP) != 0;
```

### 读取/写入 ACS Control
```c
u16 ctrl;

// 读取控制
pci_read_config_word(dev, acs_pos + PCI_ACS_CTRL, &ctrl);

// 检查是否启用
bool enabled = (ctrl & PCI_ACS_CTRL_EN) != 0;

// 启用 ACS
ctrl |= PCI_ACS_CTRL_EN;

// 写入控制
pci_write_config_word(dev, acs_pos + PCI_ACS_CTRL, ctrl);
```

### 读取/写入 ACS Extended Control
```c
u32 ext_ctrl;

// 读取扩展控制
pci_read_config_dword(dev, acs_pos + PCI_ACS_EGRESS_CTRL, &ext_ctrl);

// 检查 Egress Translation Blocking
bool egress_tl_block = (ext_ctrl & PCI_ACS_EGRESS_TL_BLOCK) != 0;

// 检查 P2P Request Blocking
bool p2p_req_block = (ext_ctrl & PCI_ACS_P2P_REQ_BLOCK) != 0;

// 检查 P2P Request Completion Blocking
bool p2p_comp_block = (ext_ctrl & PCI_ACS_P2P_COMP_BLOCK) != 0;

// 启用 Direct Translated P2P Request Blocking
bool dt_p2p_req_block = (ext_ctrl & PCI_ACS_DT_P2P_REQ_BLOCK) != 0;

// 写入扩展控制
ext_ctrl |= PCI_ACS_EGRESS_TL_BLOCK;
ext_ctrl |= PCI_ACS_P2P_REQ_BLOCK;
ext_ctrl |= PCI_ACS_P2P_COMP_BLOCK;
ext_ctrl |= PCI_ACS_DT_P2P_REQ_BLOCK;

pci_write_config_dword(dev, acs_pos + PCI_ACS_EGRESS_CTRL, ext_ctrl);
```

## ACS 功能说明

### Source Validation (SV)
- 验证请求源地址的有效性
- 防止恶意设备访问其他设备的地址

### Redirect Request (RR)
- 允许请求被重定向到 Root Complex
- 用于 P2P 拓扑优化

### Translation Blocking (TB)
- 阻止地址转换请求
- 用于 P2P 隔离

### P2P Request Blocking (PRP)
- 阻止 P2P 请求
- 用于 P2P 隔离

### P2P Completion Blocking (PCP)
- 阻止 P2P 完成请求
- 用于 P2P 隔离

### Egress Translation Blocking
- 阻止 Egress 转换中的地址转换

### Direct Translated P2P Request Blocking
- 阻止直接翻译的 P2P 请求

## 性能考虑

1. **路径检查**: `pci_acs_path_enabled()` 需要遍历整个路径
2. **设备特定配置**: 某些设备可能需要特殊的 ACS 配置
3. **P2P 支持**: ACS 对 P2P 性能有重要影响
4. **Egress Blocking**: 影响地址转换性能

## 调试信息

```c
// ACS 初始化
dev_info(&dev, "ACS capabilities: %#04x\n", caps);

// 路径检查
if (pci_acs_path_enabled(dev, NULL, PCI_ACS_CAP_RR)) {
    dev_info(&dev, "ACS Redirect Request enabled\n");
}
```

## 错误处理

1. **ACS 不支持**:
   ```c
   if (!pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ACS))
       return false;
   ```

2. **路径检查失败**:
   ```c
   if (!pci_acs_path_enabled(pdev, NULL, PCI_ACS_CAP_RR)) {
       dev_err(&dev, "ACS path check failed\n");
       return false;
   }
   ```

3. **设备特定配置失败**:
   ```c
   ret = pci_dev_specific_enable_acs(pdev);
   if (ret) {
       dev_err(&dev, "Device specific ACS enable failed: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCI_ACS`: 启用 ACS 支持
- `CONFIG_PCI_PASID`: PASID 支持（ACS 的依赖）
- `CONFIG_PCI_PRI`: PRI 支持（ACS 的依赖）

## 参考资料

- PCIe Access Control Services Specification
- PCIe Base Specification Revision 5.0+, Section 10.4
- Linux 内核源码: `drivers/pci/pci.c`
- 头文件: `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
