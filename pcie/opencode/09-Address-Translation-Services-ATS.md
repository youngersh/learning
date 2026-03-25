# Address Translation Services (ATS) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_ATS (0x0F)  
**规范参考**: PCIe ATS Specification  
**功能**: 支持 IOMMU 地址转换服务

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x0F)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    2    ATS Capability
              - Bit 0: Page Aligned Request
              - Bits [5:1]: Invalidate Queue Depth
0x06    2    ATS Control
              - Bit 0: ATS Enable
              - Bits [5:2]: STU (Smallest Translation Unit)
```

### STU 值
| 值 | 页大小 | 描述 |
|-----|--------|------|
| 0 | 保留 | 保留 |
| 1 | 4KB | 4KB 页 |
| 2 | 8KB | 8KB 页 |
| 3 | 16KB | 16KB 页 |
| 4 | 32KB | 32KB 页 |
| 5 | 64KB | 64KB 页 |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/ats.c`  
**函数**: `pci_ats_init()` (行 20-32)

```c
void pci_ats_init(struct pci_dev *dev)
{
    int pos;

    // 1. 检查 ATS 是否被禁用
    if (pci_ats_disabled())
        return;

    // 2. 查找 ATS capability
    pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ATS);
    if (!pos)
        return;

    // 3. 保存 ATS capability 位置
    dev->ats_cap = pos;
}
```

### 检查 ATS 支持

**函数**: `pci_ats_supported()` (行 41-48)

```c
bool pci_ats_supported(struct pci_dev *dev)
{
    // 1. 检查 ATS capability
    if (!dev->ats_cap)
        return false;

    // 2. 检查设备是否可信
    return (dev->untrusted == 0);
}
```

### 准备 ATS

**函数**: `pci_prepare_ats()` (行 60-81)

```c
int pci_prepare_ats(struct pci_dev *dev, int ps)
{
    u16 ctrl;

    // 1. 检查 ATS 支持
    if (!pci_ats_supported(dev))
        return -EINVAL;

    // 2. 检查是否已启用
    if (WARN_ON(dev->ats_enabled))
        return -EBUSY;

    // 3. 检查 STU 范围
    if (ps < PCI_ATS_MIN_STU)
        return -EINVAL;

    // 4. VF 不需要单独配置
    if (dev->is_virtfn)
        return 0;

    // 5. 保存 STU
    dev->ats_stu = ps;

    // 6. 配置 ATS Control
    ctrl = PCI_ATS_CTRL_STU(dev->ats_stu - PCI_ATS_MIN_STU);
    pci_write_config_word(dev, dev->ats_cap + PCI_ATS_CTRL, ctrl);

    return 0;
}
```

### 启用 ATS

**函数**: `pci_enable_ats()` (行 90-122)

```c
int pci_enable_ats(struct pci_dev *dev, int ps)
{
    u16 ctrl;
    struct pci_dev *pdev;

    // 1. 检查 ATS 支持
    if (!pci_ats_supported(dev))
        return -EINVAL;

    // 2. 检查是否已启用
    if (WARN_ON(dev->ats_enabled))
        return -EBUSY;

    // 3. 检查 STU 范围
    if (ps < PCI_ATS_MIN_STU)
        return -EINVAL;

    // 4. 处理 VF
    if (dev->is_virtfn) {
        pdev = pci_physfn(dev);
        
        // VF 必须使用与 PF 相同的 STU
        if (pdev->ats_stu != ps)
            return -EINVAL;
    } else {
        // PF 保存 STU
        dev->ats_stu = ps;
   
        
        // 配置 STU
        ctrl |= PCI_ATS_CTRL_STU(dev->ats_stu - PCI_ATS_MIN_STU);
    }

    // 5. 启用 ATS
    ctrl = PCI_ATS_CTRL_ENABLE;
    pci_write_config_word(dev, dev->ats_cap + PCI_ATS_CTRL, ctrl);

    dev->ats_enabled = 1;
    return 0;
}
```

### 禁用 ATS

**函数**: `pci_disable_ats()` (行 128-141)

```c
void pci_disable_ats(struct pci_dev *dev)
{
    u16 ctrl;

    // 1. 检查是否已启用
    if (WARN_ON(!dev->ats_enabled))
        return;

    // 2. 读取当前控制
    pci_read_config_word(dev, dev->ats_cap + PCI_ATS_CTRL, &ctrl);

    // 3. 禁用 ATS
    ctrl &= ~PCI_ATS_CTRL_ENABLE;
    pci_write_config_word(dev, dev->ats_cap + PCI_ATS_CTRL, ctrl);

    dev->ats_enabled = 0;
}
```

### 恢复 ATS 状态

**函数**: `pci_restore_ats_state()` (行 143-154)

```c
void pci_restore_ats_state(struct pci_dev *dev)
{
    u16 ctrl;

    // 1. 检查是否已启用
    if (!dev->ats_enabled)
        return;

    // 2. 配置 ATS Control
    ctrl = PCI_ATS_CTRL_ENABLE;
    if (!dev->is_virtfn)
        ctrl |= PCI_ATS_CTRL_STU(dev->ats_stu - PCI_ATS_MIN_STU);
    
    pci_write_config_word(dev, dev->ats_cap + PCI_ATSudi_CTRL, ctrl);
}
```

### 查询 Invalidate Queue Depth

**函数**: `pci_ats_queue_depth()` (行 168-180)

```c
int pci_ats_queue_depth(struct pci_dev *dev)
{
    u16 cap;

    // 1. 检查 ATS capability
    if (!dev->ats_cap)
        return -EINVAL;

    // 2. VF 不支持独立队列
    if (dev->is_virtfn)
        return 0;

    // 3. 读取 ATS Capability
    pci_read_config_word(dev, dev->ats_cap + PCI_ATS_CAP, &cap);

    // 4. 返回队列深度
    return PCI_ATS_CAP_QDEP(cap) ? PCI_ATS_CAP_QDEP(cap) : PCI_ATS_MAX_QDEP;
}
```

### 查询 Page Aligned Request

**函数**: `pci_ats_page_aligned()` (行 193-206)

```c
int pci_ats_page_aligned(struct pci_dev *pdev)
{
    u16 cap;

    // 1. 检查 ATS capability
    if (!pdev->ats_cap)
        return 0;

    // 2. 读取 ATS Capability
    pci_read_config_word(pdev, pdev->ats_cap + PCI_ATS_CAP, &cap);

    // 3. 返回 Page Aligned Request 状态
    if (cap & PCI_ATS_CAP_PAGE_ALIGNED)
        return 1;

    return 0;
}
```

## Root Port 处理

Root Port 通常不实现 ATS capability，因为它是 Bridge 设备。ATS 主要用于 Endpoint 设备与 IOMMU 交互。

## 设备驱动使用

### 1. 检查 ATS 支持
```c
if (pci_ats_supported(pdev)) {
    // 设备支持 ATS
    
    // 查询队列深度
    int qdepth = pci_ats_queue_depth(pdev);
    dev_info(&pdev->dev, "ATS queue depth: %d\n", qdepth);
    
    // 查询 Page Aligned Request
    if (pci_ats_page_aligned(pdev)) {
        dev_info(&pdev->dev, "ATS page aligned request supported\n");
    }
}
```

### 2. 启用 ATS
```c
// 选择 STU (页大小)
int ps = 2;  // 8KB 页

// 准备 ATS
int ret = pci_prepare_ats(pdev, ps);
if (ret) {
    dev_err(&pdev->dev, "Failed to prepare ATS: %d\n", ret);
    return ret;
}

// 启用 ATS
ret = pci_enable_ats(pdev, ps);
if (ret) {
    dev_err(&pdev->dev, "Failed to enable ATS: %d\n", ret);
    return ret;
}

dev_info(&pdev->dev, "ATS enabled, STU: %d\n", ps);
```

### 3. 禁用 ATS
```c
// 禁用 ATS
pci_disable_ats(pdev);
```

### 4. SR-IOV 支持
```c
// PF 启用 ATS 后才能创建 VF
if (pdev->is_physfn) {
    // 准备 ATS
    pci_prepare_ats(pdev, ps);
    
    // 启用 ATS
    pci_enable_ats(pdev, ps);
    
    // 现在可以创建 VF
    pci_enable_sriov(pdev, num_vfs);
}
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_ats_init()
                 └─> pci_find_ext_capability(PCI_EXT_CAP_ID_ATS)

IOMMU 驱动:
  └─> pci_prepare_ats()
       ├─> pci_ats_supported()  // 检查支持
       └─> pci_write_config_word()  // 配置 STU
  └─> pci_enable_ats()
       ├─> pci_write_config_word()  // 启用 ATS
       └─> 设置 dev->ats_enabled
```

## 配置空间访问

### 查找 ATS Capability
```c
int ats_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ATS);
```

### 读取 ATS Capability
```c
u16 cap;
pci_read_config_word(dev, ats_pos + PCI_ATS_CAP, &cap);

// 查询 Page Aligned Request
bool page_aligned = (cap & PCI_ATS_CAP_PAGE_ALIGNED) != 0;

// 查询 Invalidate Queue Depth
int qdepth = PCI_ATS_CAP_QDEP(cap) ? PCI_ATS_CAP_QDEP(cap) : PCI_ATS_MAX_QDEP;
```

### 读取/写入 ATS Control
```c
u16 ctrl;

// 读取控制
pci_read_config_word(dev, ats_pos + PCI_ATS_CTRL, &ctrl);

// 检查是否启用
bool enabled = (ctrl & PCI_ATS_CTRL_ENABLE) != 0;

// 获取 STU
int stu = FIELD_GET(PCI_ATS_CTRL_STU_MASK, ctrl);

// 启用 ATS
ctrl |= PCI_ATS_CTRL_ENABLE;
ctrl &= ~PCI_ATS_CTRL_STU_MASK;
ctrl |= FIELD_PREP(PCI_ATS_CTRL_STU_MASK, stu);

// 写入控制
pci_write_config_word(dev, ats_pos + PCI_ATS_CTRL,udi_ctrl);
```

## ATS 与 PRI/PASID 的关系

```
ATS → PRI → PASID

ATS: 地址转换服务
PRI: 页面请求接口
PASID: 进程地址空间 ID

通常一起使用以支持 IOMMU 功能
```

## 性能考虑

1. **STU 选择**: 较小的 STU 提供更好的性能
2. **队列深度**: 影响 invalidate 操作的吞吐量
3. **Page Aligned**: 简化地址转换
4. **VF 共享**: VF 共享 PF 的 ATS 配置

## 调试信息

```c
// ATS 启用
dev_info(&pdev->dev, "ATS enabled, STU: %d\n", ps);

// 队列深度
dev_info(&pdev->dev, "ATS queue depth: %d\n", qdepth);

// Page Aligned
if (pci_ats_page_aligned(pdev)) {
    dev_info(&pdev->dev, "ATS page aligned request supported\n");
}
```

## 错误处理

1. **ATS 不支持**:
   ```c
   if (!dev->ats_cap)
       return false;
   ```

2. **设备不可信**:
   ```c
   if (dev->untrusted)
       return false;
   ```

3. **STU 无效**:
   ```c
   if (ps < PCI_ATS_MIN_STU)
       return -EINVAL;
   ```

4. **启用失败**:
   ```c
   ret = pci_enable_ats(pdev, ps);
   if (ret) {
       dev_err(&pdev->dev, "Failed to enable ATS: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCI_ATS`: 启用 ATS 支持
- `CONFIG_PCI_PRI`: PRI 支持（ATS 的依赖）
- `CONFIG_PCI_PASID`: PASID 支持（ATS 的依赖）

## 参考资料

- PCIe Address Translation Services Specification
- PCIe Base Specification Revision 5.0+, Section 10.5
- Linux 内核源码: `drivers/pci/ats.c`
- 头文件: `include/linux/pci-ats.h`, `include/uapi/linux/pci_regs.h`
