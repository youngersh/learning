# Single Root I/O Virtualization (SR-IOV) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_SRIOV (0x10)  
**规范参考**: PCIe SR-IOV Specification  
**功能**: 允许单个 PF (Physical Function) 创建多个 VF (Virtual Functions)

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x10)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    4    SR-IOV Capabilities
              - Bits [15:0]: Total VFs
              - Bits [23:16]: Initial VFs
              - Bit 24: VF Migration Capable
              - Bit 25: ARI Hierarchical Capable
              - Bit 26: ARI Capable
              - Bit 27: VF Migration Interrupt
0x08    2    SR-IOV Control
              - Bit[0]: VF Enable
              - Bit[1]: VF Migration Enable
              - Bit[2]: VF Migration Interrupt Ack
              - Bit[3]: MSE (Memory Space Enable)
              - Bits [7:4]: Reserved
              - Bits [15:8]: VF Migration State
0x0A    2    SR-IOV Status
              - Bit[0]: VF Migration Status
0x0C    2    Initial VF Offset
0x0E    2    VF Stride
0x10    2    VF Device ID
0x12    4    Supported Page Sizes
0x16    4    VF Migration State Array Offset
0x1A    4    VF Migration State Array Size
0x1E    N    VF BAR 0-5 (each 4 bytes)
```

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/iov.c`  
**函数**: `pci_iov_init()` (行 990-1002)

```c
int pci_iov_init(struct pci_dev *dev)
{
    int pos;

    // 1. 检查是否是 PCIe 设备
    if (!pci_is_pcie(dev))
        return -ENODEV;

    // 2. 查找 SR-IOV capability
    pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_SRIOV);
    if (pos)
        return sriov_init(dev, pos);

    return -ENODEV;
}
```

### SR-IOV 初始化

**函数**: `sriov_init()` (内部函数)

```c
static int sriov_init(struct pci_dev *dev, int pos)
{
    int i, bar64;
    int nres;
    u32 pgsz;
    u16 ctrl, total, initial, stride;
    struct pci_sriov *iov;

    // 1. 读取 SR-IOV Capabilities
    pci_read_config_word(dev, pos + PCI_SRIOV_CTRL, &ctrl);
    pci_read_config_word(dev, pos + PCI_SRIOV_TOTAL_VF, &total);
    pci_read_config_word(dev, pos + PCI_SRIOV_INITIAL_VF, &initial);

    // 2. 检查 Total VFs
    if (!total)
        return 0;

    // 3. 分配 SR-IOV 结构
    iov = kzalloc(sizeof(*iov), GFP_KERNEL);
    if (!iov)
        return -ENOMEM;

    // 4. 填充基本信息
    iov->pos = pos;
    iov->cap = ctrl;
    iov->total_VFs = total;
    iov->initial_VFs = initial;
    dev->is_physfn = true;

    // 5. 读取 VF Offset 和 Stride
    pci_read_config_word(dev, pos + PCI_SRIOV_VF_OFFSET, &iov->offset);
    pci_read_config_word(dev, pos + PCI_SRIOV_VF_STRIDE, &iov->stride);

    // 6. 读取 VF Device ID
    pci_read_config_word(dev, pos + PCI_SRIOV_VF_DID, &iov->vf_device);

    // 7. 读取 Page Size
    pci_read_config_dword(dev, pos + PCI_SRIOV_SUP_PGSIZE, &pgsz);
    iov->pgsz = pgsz & PCI_SRIOV_SUP_PGSIZE_MASK;

    // 8. 解析 VF BARs
    for (i = 0; i < PCI_SRIOV_NUM_BARS; i++) {
        bar64 = pci_read_config_dword(dev, pos + PCI_SRIOV_BAR + i * 4);
        if (bar64 & PCI_BASE_ADDRESS_MEM_TYPE_64)
            iov->barsz[i] = ~(bar64 & PCI_BASE_ADDRESS_MEM_MASK);
        else
            iov->barsz[i] = ~(bar64 & PCI_BASE_ADDRESS_MEM_MASK) + 1;
    }

    // 9. 设置 Function Dependency Link
    pci_read_config_byte(dev, pos + PCI_SRIOV_FUNC_LINK, &iov->link);

    // 10. 保存到设备结构
    dev->sriov = iov;

    // 11. 初始化启用 VF
    iov->num_VFs = initial;

    return 0;
}
```

### 启用 VFs

**函数**: `pci_enable_sriov()`

```c
int pci_enable_sriov(struct pci_dev *dev, int num_vfs)
{
    struct pci_sriov *iov = dev->sriov;
    int rc;

    // 1. 检查是否是 PF
    if (!dev->is_physfn)
        return -ENODEV;

    // 2. 检查 VF 数量
    if (num_vfs > iov->total_VFs)
        return -EINVAL;

    // 3. 启用 Memory Space
    pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL,
                      PCI_SRIOV_CTRL_MSE);

    // 4. 设置 VF 数量
    pci_write_config_word(dev, iov->pos + PCI_SRIOV_NUM_VF, num_vfs);

    // 5. 启用 VFs
    pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL,
                      PCI_SRIOV_CTRL_VFE | PCI_SRIOV_CTRL_MSE);

    // 6. 等待 VFs 可用
    msleep(100);

    // 7. 扫描 VFs
    rc = pci_iov_add_virtfn(dev);
    if (rc)
        return rc;

    iov->num_VFs = num_vfs;
    return 0;
}
```

### 禁用 VFs

**函数**: `pci_disable_sriov()`

```c
void pci_disable_sriov(struct pci_dev *dev)
{
    struct pci_sriov *iov = dev->sriov;

    // 1. 检查是否是 PF
    if (!dev->is_physfn)
        return;

    // 2. 移除所有 VFs
    pci_iov_remove_virtfn(dev);

    // 3. 禁用 VFs
    pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL, 0);

    // 4. 禁用 Memory Space
    pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL, 0);

    iov->num_VFs = 0;
}
```

### VF BAR 更新

**函数**: `pci_iov_update_resource()` (行 1037-1086)

```c
void pci_iov_update_resource(struct pci_dev *dev, int resno)
{
    struct pci_sriov *iov = dev->is_physfn ? dev->sriov : NULL;
    struct resource *res = pci_resource_n(dev, resno);
    int vf_bar = pci_resource_num_to_vf_bar(resno);
    struct pci_bus_region region;
    u16 cmd;
    u32 new;
    int reg;

    // 1. 检查是否是 PF
    if (!iov)
        return;

    // 2. 检查 VFs 是否已启用
    pci_read_config_word(dev, iov->pos + PCI_SRIOV_CTRL, &cmd);
    if ((cmd & PCI_SRIOV_CTRL_VFE) && (cmd & PCI_SRIOV_CTRL_MSE)) {
        dev_WARN(&dev->dev, "can't update enabled VF BAR%d %pR\n",
                 vf_bar, res);
        return;
    }

    // 3. 检查资源
    if (!res->flags)
        return;

    if (res->flags & IORESOURCE_UNSET)
        return;

    if (res->flags & IORESOURCE_PCI_FIXED)
        return;

    // 4. 写入 BAR
    pcibios_resource_to_bus(dev->bus, &region, res);
    new = region.start;
    new |= res->flags & ~PCI_BASE_ADDRESS_MEM_MASK;

    reg = iov->pos + PCI_SRIOV_BAR_BAR + 4 * vf_bar;
    pci_write_config_dword(dev, reg, new);

    // 5. 处理 64 位 BAR
    if (res->flags & IORESOURCE_MEM_64) {
        new = region.start >> 16 >> 16;
        pci_write_config_dword(dev, reg + 4, new);
    }
}
```

## Root Port 处理

Root Port 本身不实现 SR-IOV，但它需要正确处理 PF 和 VF 的枚举和配置：

```c
// Root Port 不需要特殊处理
// VFs 通过标准 PCI 枚举发现
```

## 设备驱动使用

### 1. PF 驱动

```c
// 检查是否是 PF
if (pdev->is_physfn) {
    struct pci_sriov *iov = pdev->sriov;
    
    // 获取 Total VFs
    int total_vfs = iov->total_VFs;
    
    // 获取 Initial VFs
    int initial_vfs = iov->initial_VFs;
    
    // 获取当前启用的 VFs
    int num_vfs = iov->num_VFs;
    
    dev_info(&pdev->dev, "PF: Total VFs=%d, Initial VFs=%d, Current VFs=%d\n",
             total_vfs, initial_vfs, num_vfs);
}
```

### 2. 启用 VFs

```c
// 启用指定数量的 VFs
int num_vfs = 8;
int ret = pci_enable_sriov(pdev, num_vfs);
if (ret) {
    dev_err(&pdev->dev, "Failed to enable VFs: %d\n", ret);
    return ret;
}

dev_info(&pdev->dev, "Enabled %d VFs\n", num_vfs);
```

### 3. 禁用 VFs

```c
// 禁用所有 VFs
pci_disable_sriov(pdev);
```

### 4. VF 驱动

```c
// 检查是否是 VF
if (pdev->is_virtfn) {
    // 获取 PF
    struct pci_dev *pf = pci_physfn(pdev);
    
    // 获取 VF 索引
    int vf_index = pci_vf_id(pdev);
    
    dev_info(&pdev->dev, "VF: Index=%d, PF=%s\n",
             vf_index, dev_name(&pf->dev));
}
```

### 5. VF BAR 访问

```c
// VF 使用标准的 BAR 访问
for (i = 0; i < 6; i++) {
    if (pci_resource_len(pdev, i)) {
        void __iomem *base;
        
        base = pci_iomap(pdev, i, 0);
        if (!base) {
            dev_err(&pdev->dev, "Failed to map BAR %d\n", i);
            continue;
        }
        
        // 使用 BAR
        // ...
        
        pci_iounmap(base);
    }
}
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_iov_init()
                 └─> sriov_init()
                       ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_SRIOV)
                       ├─> kzalloc()  // 分配 sriov 结构
                       ├─> pci_read_config_word()  // 读取 capabilities
                       ├─> pci_read_config_word()  // 读取 total/initial VFs
                       ├─> pci_read_config_word()  // 读取 offset/stride
                       ├─> pci_read_config_dword()  // 读取 page size
                       └─> pci_read_config_dword()  // 读取 VF BARs

PF 驱动:
  └─> pci_enable_sriov()
       ├─> pci_write_config_word()  // 启用 MSE
       ├─> pci_write_config_word()  // 设置 num_vfs
       ├─> pci_write_config_word()  // 启用 VFE
       └─> pci_iov_add_virtfn()  // 扫描 VFs
```

## 配置空间访问

### 查找 SR-IOV Capability
```c
int sriov_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_SRIOV);
```

### 读取 SR-IOV Capabilities
```c
u16 total_vfs, initial_vfs;
u32 caps;

pci_read_config_word(dev, sriov_pos + PCI_SRIOV_TOTAL_VF, &total_vfs);
pci_read_config_word(dev, sriov_pos + PCI_SRIOV_INITIAL_VF, &initial_vfs);
pci_read_config_dword(dev, sriov_pos + PCI_SRIOV_CAP, &caps);

bool vf_migration = (caps & PCI_SRIOV_CAP_VFM) != 0;
bool ari_capable = (caps & PCI_SRIOV_CAP_ARI) != 0;
```

### 读取 VF Offset 和 Stride
```c
u16 offset, stride;

pci_read_config_word(dev, sriov_pos + PCI_SRIOV_VF_OFFSET, &offset);
pci_read_config_word(dev, sriov_pos + PCI_SRIOV_VF_STRIDE, &stride);
```

### 读取 VF BAR
```c
u32 bar0, bar1;

pci_read_config_dword(dev, sriov_pos + PCI_SRIOV_BAR, &bar0);
pci_read_config_dword(dev, sriov_pos + PCI_SRIOV_BAR + 4, &bar1);

// 计算 BAR 大小
resource_size_t size0 = ~(bar0 & PCI_BASE_ADDRESS_MEM_MASK) + 1;
resource_size_t size1 = ~(bar1 & PCI_BASE_ADDRESS_MEM_MASK) + 1;
```

### 启用 VFs
```c
// 1. 启用 Memory Space
pci_write_config_word(dev, sriov_pos + PCI_SRIOV_CTRL,
                      PCI_SRIOV_CTRL_MSE);

// 2. 设置 VF 数量
pci_write_config_word(dev, sriov_pos + PCI_SRIOV_NUM_VF, num_vfs);

// 3. 启用 VFs
pci_write_config_word(dev, sriov_pos + PCI_SRIOV_CTRL,
                      PCI_SRIOV_CTRL_VFE | PCI_SRIOV_CTRL_MSE);
```

## VF 枚举

```
PF 启用 VFs
    ↓
系统扫描 PCI 总线
    ↓
发现 VF 设备
    ↓
为每个 VF 创建 pci_dev
    ↓
调用 pci_enable_sriov()
    ↓
VF 可用
```

## 性能考虑

1. **VF 数量**: 每个 PF 最多支持 256 个 VF
2. **BAR 对齐**: VF BAR 必须按页对齐
3. **内存开销**: 每个 VF 需要独立的配置空间和资源
4. **MSI 共享**: VF 通常共享 PF 的 MSI-X

## 调试信息

```c
// PF 信息
dev_info(&pdev->dev, "PF: Total VFs=%d, Initial VFs=%d\n",
         iov->total_VFs, iov->initial_VFs);

// VF 信息
dev_info(&pdev->dev, "VF: Index=%d, PF=%s\n",
         pci_vf_id(pdev), dev_name(&pf->dev));
```

## 错误处理

1. **SR-IOV 不支持**:
   ```c
   if (!pci_find_ext_capability(dev, PCI_EXT_CAP_ID_SRIOV))
       return -ENODEV;
   ```

2. **VF 数量超限**:
   ```c
   if (num_vfs > iov->total_VFs)
       return -EINVAL;
   ```

3. **启用失败**:
   ```c
   ret = pci_enable_sriov(pdev, num_vfs);
   if (ret) {
       dev_err(&pdev->dev, "Failed to enable VFs: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCI_IOV`: 启用 SR-IOV 支持
- `CONFIG_PCI_IOV_VGA`: VGA 设备的 SR-IOV 支持
- `max_vfs`: 系统参数，限制最大 VF 数量

## 参考资料

- PCIe Single Root I/O Virtualization Specification
- PCIe Base Specification Revision 5.0+, Section 9.5
- Linux 内核源码: `drivers/pci/iov.c`
- 头文件: `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
