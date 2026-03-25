# MSI-X (Message Signaled Interrupts Extended) Capability 详细分析

## 概述

**Capability ID**: PCI_CAP_ID_MSIX (0x11)  
**规范参考**: PCI 3.0+, PCIe 1.0+  
**功能**: 扩展的 MSI，支持最多 2048 个中断向量

## PCIe SPEC 定义

### Capability 结构
```
Offset  Size  Description
0x00    1    Capability ID (0x11)
0x01    1    Next Capability Pointer
0x02    2    Message Control
              - Bits [10:0]: Table Size (BIR + offset)
              - Bit 14: Function Mask
              - Bit 15: MSI-X Enable
0x04    4    Message Address Table (BAR Indicator + Offset)
0x08    4    Pending Bit Array (BAR Indicator + Offset)
```

### MSI-X Table Entry 结构
存储在设备内存空间中，每个 entry 16 字节：
```
Offset  Size  Description
0x00    4    Message Address Lower
0x04    4    Message Address Upper
0x08    4    Message Data
0x0C    4    Vector Control
              - Bit 0: Mask Bit
              - Bits [31:16]: Steering Tag (optional)
```

### BAR Indicator 编码
| 值 | BAR |
|-----|-----|
| 0 | BAR 0 |
| 1 | BAR 1 |
| 2 | BAR 2 |
| 3 | BAR 3 |
| 4 | BAR 4 |
| 5 | BAR 5 |
| 6-7 | ROM |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/msi/pcidev_msi.c`  
**函数**: `pci_msix_init()` (行 30-43)

```c
void pci_msix_init(struct pci_dev *dev)
{
    u16 ctrl;

    // 1. 查找 MSI-X capability
    dev->msix_cap = pci_find_capability(dev, PCI_CAP_ID_MSIX);
    if (!dev->msix_cap)
        return;

    // 2. 读取 Message Control
    pci_read_config_word(dev, dev->msix_cap + PCI_MSIX_FLAGS, &ctrl);

    // 3. 如果已启用，禁用它
    if (ctrl & PCI_MSIX_FLAGS_ENABLE) {
        pci_write_config_word(dev, dev->msix_cap + PCI_MSIX_FLAGS,
                      ctrl & ~PCI_MSIX_FLAGS_ENABLE);
    }
}
```

### 启用 MSI-X

**文件**: `drivers/pci/msi/msi.c`

#### 1. pci_enable_msix()
```c
int pci_enable_msix(struct pci_dev *dev, struct msi_desc *entry,
                  int nvec)
{
    u16 ctrl;
    unsigned long flags;
    int offset, entries;
    void __iomem *base;

    // 1. 检查 MSI-X 支持
    if (!pci_msix_supported(dev))
        return -EINVAL;

    // 2. 获取 MSI-X table 信息
    pci_read_config_word(dev, dev->msix_cap + PCI_MSIX_FLAGS, &ctrl);
    entries = 1 << ((ctrl & PCI_MSIX_FLAGS_QSIZE) + 1);

    // 3. 映射 MSI-X table
    offset = pci_msix_table_offset(dev);
    base = pci_msix_map_table(dev, offset);

    // 4. 配置每个向量
    for (i = 0; i < nvec; i++) {
        // 写入地址
        writel(entry[i].msg.address_lo, base + i * 16);
        writel(entry[i].msg.address_hi, base + i * 16 + 4);
        
        // 写入数据
        writel(entry[i].msg.data, base + i * 16 + 8);
        
        // 取消屏蔽
        writel(0, base + i * 16 + 12);
    }

    // 5. 启用 MSI-X
    pci_read_config_word(dev, dev->msix_cap + PCI_MSIX_FLAGS, &ctrl);
    ctrl |= PCI_MSIX_FLAGS_ENABLE;
    ctrl &= ~PCI_MSIX_FLAGS_MASKALL;
    pci_write_config_word(dev, dev->msix_cap + PCI_MSIX_FLAGS, ctrl);

    dev->msix_enabled = 1;
    return i;
}
```

#### 2. pci_msix_setup_msi_irqs()
```c
int pci_msix_setup_msi_irqs(struct pci_dev *dev, struct msi_desc *entries,
                           int nvec)
{
    struct irq_domain *domain;
    int ret;

    // 1. 获取 MSI-X domain
    domain = pci_msix_get_domain(dev);
    if (!domain)
        return -ENOTSUPP;

    // 2. 分配 IRQ
    ret = msi_domain_alloc_irqs(domain, entries, nvec, NULL);
    if (ret)
        return ret;

    // 3. 配置 MSI-X table
    ret = pci_msix_setup_entries(dev, entries, nvec);
    if (ret)
        goto out;

    // 4. 启用 MSI-X
    pci_msix_enable(dev, entries, nvec);

    return 0;

out:
    msi_domain_free_irqs(domain, entries, nvec);
    return ret;
}
```

### 禁用 MSI-X

```c
void pci_disable_msix(struct pci_dev *dev)
{
    u16 ctrl;

    if (!dev->msix_enabled)
        return;

    // 1. 禁用 MSI-X
    pci_read_config_word(dev, dev->msix_cap + PCI_MSIX_FLAGS, &ctrl);
    ctrl &= ~PCI_MSIX_FLAGS_ENABLE;
    ctrl |= PCI_MSIX_FLAGS_MASKALL;
    pci_write_config_word(dev, dev->msix_cap + PCI_MSIX_FLAGS, ctrl);

    // 2. 解映射表
    if (dev->msix_entries) {
        pci_msix_unmap_table(dev);
        dev->msix_entries = NULL;
    }

    dev->msix_enabled = 0;
}
```

### 屏蔽/启用向量

```c
void pci_msix_mask_irq(struct pci_dev *dev, int irq)
{
    void __iomem *base = dev->msix_entries;
    int offset = irq * 16 + 12;
    u32 ctrl;

    ctrl = readl(base + offset);
    ctrl |= 1;
    writel(ctrl, base + offset);
}

void pci_msix_unmask_irq(struct pci_dev *dev, int irq)
{
    void __iomem *base = dev->msix_entries;
    int offset = irq * 16 + 12;
    u32 ctrl;

    ctrl = readl(base + offset);
    ctrl &= ~1;
    writel(ctrl, base + offset);
}
```

### 映射 MSI-X Table

```c
void __iomem *pci_msix_map_table(struct pci_dev *dev, int offset)
{
    resource_size_t phys_addr;
    unsigned long size;
    void __iomem *base;
    int bar;

    // 1. 获取 BAR 和偏移
    bar = (offset & PCI_MSIX_TABLE_BIR);
    offset = offset & ~PCI_MSIX_TABLE_BIR;

    // 2. 计算物理地址
    phys_addr = pci_resource_start(dev, bar) + offset;

    // 3. 计算表大小
    size = pci_msix_table_size(dev);

    // 4. 映射
    base = ioremap(phys_addr, size);
    if (!base)
        return NULL;

    return base;
}
```

## Root Port 处理

Root Port 通常不使用 MSI-X，因为它们使用 PCIe 原生中断。如果 Root Port 支持 MSI-X，可以像 Endpoint 一样配置。

## 设备驱动使用

### 1. 请求 MSI-X 中断
```c
// 最小/最大向量数
int min_vecs = 1;
int max_vecs = pci_msix_vec_count(pdev);

// 分配
int ret = pci_alloc_irq_vectors(pdev, min_vecs, max_vec_vecs, NULL);
if (ret < 0) {
    dev_err(&pdev->dev, "Failed to allocate MSI-X: %d\n", ret);
    return ret;
}

// 使用分配的向量数
int nvec = ret;
```

### 2. 获取 IRQ 号
```c
for (i = 0; i < nvec; i++) {
    int irq = pci_irq_vector(pdev, i);
    request_irq(irq, handlers[i], IRQF_SHARED, 
               "my_driver", pdev);
}
```

### 3. 释放 MSI-X
```c
pci_free_irq_vectors(pdev);
```

### 4. 检查 MSI-X 支持
```c
if (pdev->msix_cap) {
    // 设备支持 MSI-X
    int table_size = pci_msix_vec_count(pdev);
    dev_info(&pdev->dev, "MSI-X table size: %d\n", table_size);
}
```

### 5. 使用 affinity
```c
struct irq_affinity_desc affinity;
init_irq_affinity_desc(&affinity, nvec);

// 设置每个向量的 CPU affinity
for (i = 0; i < nvec; i++) {
    cpumask_set_cpu(i, &affinity.mask[i]);
}

// 分配带 affinity 的向量
int ret = pci_alloc_irq_vectors(pdev, 1, nvec, &affinity);
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_msix_init()
                 ├─> pci_find_capability(PCI_CAP_ID_MSIX)
                 ├─> pci_read_config_word()  // 读取 Message Control
                 └─> pci_write_config_word()  // 禁用 MSI-X（如果已启用）

驱动调用:
  └─> pci_alloc_irq_vectors()
       └─> pci_msix_setup_msi_irqs()
            ├─> msi_domain_alloc_irqs()  // 分配 IRQ
            ├─> pci_msix_map_table()  // 映射 MSI-X table
            ├─> writel()  // 配置每个 entry
            └─> pci_write_config_word()  // 启用 MSI-X
```

## 配置空间访问

### 查找 MSI-X Capability
```c
int msix_pos = pci_find_capability(dev, PCI_CAP_ID_MSIX);
```

### 读取 Message Control
```c
u16 ctrl;
pci_read_config_word(dev, msix_pos + PCI_MSIX_FLAGS, &ctrl);

// 表大小
int table_size = 1 << ((ctrl & PCI_MSIX_FLAGS_QSIZE) + 1);

// 检查功能
bool has_masking = (ctrl & PCI_MSIX_FLAGS_MASKALL) != 0;
bool is_enabled = (ctrl & PCI_MSIX_FLAGS_ENABLE) != 0;
```

### 读取 Table Offset
```c
u32 table_offset;
pci_read_config_dword(dev, msix_pos + PCI_MSIX_TABLE, &table_offset);

int bar = table_offset & PCI_MSIX_TABLE_BIR;
u32 offset = table_offset & ~PCI_MSIX_TABLE_BIR;
```

### 启用 MSI-X
```c
u16 ctrl;
pci_read_config_word(dev, msix_pos + PCI_MSIX_FLAGS, &ctrl);
ctrl |= PCI_MSIX_FLAGS_ENABLE;
ctrl &= ~PCI_MSIX_FLAGS_MASKALL;  // 取消全局屏蔽
pci_write_config_word(dev, msix_pos + PCI_MSIX_FLAGS, ctrl);
```

### 屏蔽所有向量
```c
u16 ctrl;
pci_read_config_word(dev, msix_pos + PCI_MSIX_FLAGS, &ctrl);
ctrl |= PCI_MSIX_FLAGS_MASKALL;
pci_write_config_word(dev, msix_pos + PCI_MSIX_FLAGS, ctrl);
```

## MSI-X Table 配置

### 写入 Table Entry
```c
void __iomem *table = dev->msix_entries;
int entry = irq;

// 写入地址
writel(address_lo, table + entry * 16);
writel(address_hi, table + entry * 16 + 4);

// 写入数据
writel(data, table + entry * 16 + 8);

// 屏蔽/启用
writel(mask_bit, table + entry * 16 + 12);
```

### 读取 Pending Bit Array
```c
u32 pba_offset;
pci_read_config_dword(dev, msix_pos + PCI_MSIX_PBA, &pba_offset);

int bar = pba_offset & PCI_MSIX_PBA_BIR;
u32 offset = pba_offset & ~PCI_MSIX_PBA_BIR;

void __iomem *pba = ioremap(pci_resource_start(dev, bar) + offset, 4);
u32 pending = readl(pba);
```

## MSI-X 与 MSI 的区别

| 特性 | MSI | MSI-X |
|------|-----|-------|
| 最大向量数 | 32 | 2048 |
| 表大小 | 固定 | 可配置 |
| 地址 | 配置空间 | 内存映射表 |
| 数据 | 配置空间 | 内存映射表 |
| 屏蔽 | 可选 | 每向量独立 |
| 掩序 | 简单 | 复杂 |
| 共享 | 否 | 否 |
| 掩序 | 无 | Steering Tag |

## 性能考虑

1. **Table 映射**: MSI-X table 需要映射到内核地址空间
2. **Cache 一致性**: 确保 table 的写入对设备可见
3. **Pending Bit Array**: 用于检查挂起的中断
4. **Affinity**: 支持每个向量的 CPU 亲和性

## 调试信息

```c
// 在驱动中
dev_info(&pdev->dev, "Allocated %d MSI-X vectors\n", nvec);

// 检查表大小
int table_size = pci_msix_vec_count(pdev);
dev_info(&pdev->dev, "MSI-X table size: %d\n", table_size);
```

## 错误处理

1. **MSI-X 不支持**:
   ```c
   if (!dev->msix_cap)
       return -EINVAL;
   ```

2. **Table 映射失败**:
   ```c
   base = ioremap(phys_addr, size);
   if (!base) {
       dev_err(&pdev->dev, "Failed to map MSI-X table\n");
       return -ENOMEM;
   }
   ```

3. **分配失败**:
   ```c
   ret = pci_alloc_irq_vectors(pdev, 1, nvec, NULL);
   if (ret < 0) {
       dev_err(&pdev->dev, "Failed to allocate MSI-X: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCI_MSI`: 启用 MSI 支持
- `CONFIG_PCI_MSI_IRQ_DOMAIN`: MSI IRQ domain
- `CONFIG_IRQ_REMAP`: IRQ 重映射支持

## 参考资料

- PCI Local Bus Specification 3.0+, Section 6.8.2
- PCIe Base Specification Revision 1.0+, Section 6.8.2
- Linux 内核源码: `drivers/pci/msi/`
- 头文件: `include/linux/msi.h`, `include/uapi/linux/pci_regs.h`
