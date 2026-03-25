# MSI (Message Signaled Interrupts) Capability 详细分析

## 概述

**Capability ID**: PCI_CAP_ID_MSI (0x05)  
**规范参考**: PCI 2.2+, PCIe 1.0+  
**功能**: 使用内存写事务替代传统的 INTx 中断

## PCIe SPEC 定义

### Capability 结构
```
Offset  Size  Description
0x00    1    Capability ID (0x05)
0x01    1    Next Capability Pointer
0x02    2    Message Control
              - Bit 0: MSI Enable
              - Bits [3:1]: Multiple Message Capable (MMC)
              - Bits [6:4]: Multiple Message Enable (MME)
              - Bit 7: 64-bit Address Capable
              - Bit 8: Per-vector Masking Capable
0x04    4    Message Address (Lower 32 bits)
0x08    2    Message Data (16-bit)
0x08    4    Message Address (Upper 32 bits, if 64-bit)
0x0C    2    Message Data (16-bit, if 64-bit)
0x0C    4    Mask Bits (optional, if Per-vector Masking)
0x10    4    Pending Bits (optional, if Per-vector Masking)
```

### Multiple Message Encoding
| 值 | 消息数量 |
|-----|----------|
| 000 | 1 |
| 001 | 2 |
| 010 | 4 |
| 011 | 8 |
| 100 | 16 |
| 101 | 32 |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/msi/pcidev_msi.c`  
**函数**: `pci_msi_init()` (行 12-28)

```c
void pci_msi_init(struct pci_dev *dev)
{
    u16 ctrl;

    // 1. 查找 MSI capability
    dev->msi_cap = pci_find_capability(dev, PCI_CAP_ID_MSI);
    if (!dev->msi_cap)
        return;

    // 2. 读取 Message Control
    pci_read_config_word(dev, dev->msi_cap + PCI_MSI_FLAGS, &ctrl);

    // 3. 如果已启用，禁用它
    if (ctrl & PCI_MSI_FLAGS_ENABLE) {
        pci_write_config_word(dev, dev->msi_cap + PCI_MSI_FLAGS,
                      ctrl & ~PCI_MSI_FLAGS_ENABLE);
    }

    // 4. 记录 64 位支持情况
    if (!(ctrl & PCI_MSI_FLAGS_64BIT))
        dev->no_64bit_msi = 1;
}
```

### 启用 MSI

**文件**: `drivers/pci/msi/msi.c`

#### 1. pci_enable_msi()
```c
int pci_enable_msi(struct pci_dev *dev, struct msi_desc *entry)
{
    u16 control;
    unsigned long flags;

    // 1. 检查 MSI 支持
    if (!pci_msi_supported(dev, entry))
        return -EINVAL;

    // 2. 分配 MSI 向量
    entry->msi_attrib.is_msix = 0;
    flags = entry->msi_attrib.is_64 ? PCI_BASE_ADDR_MEM_TYPE_64 :
             PCI_BASE_ADDR_MEM_TYPE_32;
    
    // 3. 配置 MSI 地址和数据
    pci_write_config_dword(dev, dev->msi_cap + PCI_MSI_ADDRESS_LO,
                       entry->msg.address_lo);
    if (entry->msi_attrib.is_64) {
        pci_write_config_dword(dev, dev->msi_cap + PCI_MSI_ADDRESS_HI,
                           entry->msg.address_hi);
        pci_write_config_word(dev, dev->msi_cap + PCI_MSI_DATA_64,
                           entry->msg.data);
    } else {
        pci_write_config_word(dev, dev->msi_cap + PCI_MSI_DATA_32,
                           entry->msg.data);
    }

    // 4. 启用 MSI
    pci_read_config_word(dev, dev->msi_cap + PCI_MSI_FLAGS, &control);
    control |= PCI_MSI_FLAGS_ENABLE;
    pci_write_config_word(dev, dev->msi_cap + PCI_MSI_FLAGS, control);

    dev->msi_enabled = 1;
    return 0;
}
```

#### 2. pci_msi_setup_msi_irqs()
```c
int pci_msi_setup_msi_irqs(struct pci_dev *dev, int nvec,
                          const struct irq_affinity_desc *affdesc)
{
    struct irq_domain *domain;
    struct msi_desc *entries;
    int ret;

    // 1. 获取 MSI domain
    domain = pci_msi_get_domain(dev);
    if (!domain)
        return -ENOTSUPP;

    // 2. 分配 MSI 描述符
    entries = alloc_msi_entries(dev, nvec);
    if (!entries)
        return -ENOMEM;

    // 3. 分配 IRQ
    ret = msi_domain_alloc_irqs(domain, entries, nvec, affdesc);
    if (ret)
        goto out;

    // 4. 配置 MSI
    ret = pci_msi_setup_msi_irq(dev, entries, nvec);
    if (ret)
        goto out;

    return 0;

out:
    free_msi_entries(entries, nvec);
    return ret;
}
```

### 禁用 MSI

```c
void pci_disable_msi(struct pci_dev *dev)
{
    u16 control;

    if (!dev->msi_enabled)
        return;

    // 1. 读取当前控制
    pci_read_config_word(dev, dev->msi_cap + PCI_MSI_FLAGS, &control);

    // 2. 清除使能位
    control &= ~PCI_MSI_FLAGS_ENABLE;
    pci_write_config_word(dev, dev->msi_cap + PCI_MSI_FLAGS, control);

    // 3. 清除地址和数据
    pci_write_config_dword(dev, dev->msi_cap + PCI_MSI_ADDRESS_LO, 0);
    if (dev->no_64bit_msi == 0)
        pci_write_config_dword(dev, dev->msi_cap + PCI_MSI_ADDRESS_HI, 0);
    pci_write_config_word(dev, dev->msi_cap + PCI_MSI_DATA_32, 0);

    dev->msi_enabled = 0;
}
```

### Per-vector Masking

如果设备支持 Per-vector Masking，可以单独屏蔽/启用每个向量：

```c
void pci_msi_mask_irq(struct pci_dev *dev, int irq)
{
    u32 mask_bits;
    int offset = dev->msi_cap + PCI_MSI_MASK_BITS;
    int pos = irq % 32;

    pci_read_config_dword(dev, offset + (irq / 32) * 4, &mask_bits);
    mask_bits |= (1 << pos);
    pci_write_config_dword(dev, offset + (irq / 32) * 4, mask_bits);
}

void pci_msi_unmask_irq(struct pci_dev *dev, int irq)
{
    u32 mask_bits;
    int offset = dev->msi_cap + PCI_MSI_MASK_BITS;
    int pos = irq % 32;

    pci.read_config_dword(dev, offset + (irq / 32) * 4, &mask_bits);
    mask_bits &= ~(1 << pos);
    pci_write_config_dword(dev, offset + (irq / 32) * 4, mask_bits);
}
```

## Root Port 处理

Root Port 通常不使用 MSI，因为它们使用 PCIe 原生中断（AER、PME 等）。如果 Root Port 支持 MSI，可以像 Endpoint 一样配置。

## 设备驱动使用

### 1. 请求 MSI 中断
```c
// 单个 MSI
int ret = pci_alloc_irq_vectors(pdev, 1, 1, NULL);
if (ret < 0) {
    dev_err(&pdev->dev, "Failed to allocate MSI: %d\n", ret);
    return ret;
}

// 多个 MSI
int nvec = pci_msi_vec_count(pdev);
ret = pci_alloc_irq_vectors(pdev, 1, nvec, NULL);
if (ret < 0) {
    dev_err(&pdev->dev, "Failed to allocate MSIs: %d\n", ret);
    return ret;
}
```

### 2. 获取 IRQ 号
```c
int irq = pci_irq_vector(pdev, 0);
request_irq(irq, handler, IRQF_SHARED, "my_driver", pdev);
```

### 3. 释放 MSI
```c
pci_free_irq_vectors(pdev);
```

### 4. 检查 MSI 支持
```c
if (pci_dev_msi_enabled(pdev)) {
    // MSI 已启用
}

if (pdev->msi_cap) {
    // 设备支持 MSI
}
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_msi_init()
                 ├─> pci_find_capability(PCI_CAP_ID_MSI)
                 ├─> pci_read_config_word()  // 读取 Message Control
                 ├─> pci_write_config_word()  // 禁用 MSI（如果已启用）
                 └─> 设置 dev->no_64bit_msi

驱动调用:
  └─> pci_alloc_irq_vectors()
       └─> pci_msi_setup_msi_irqs()
            ├─> msi_domain_alloc_irqs()  // 分配 IRQ
            ├─> pci_write_config_dword()  // 写入地址
            ├─> pci_write_config_word()  // 写入数据
            └─> pci_write_config_word()  // 启用 MSI
```

## 配置空间访问

### 查找 MSI Capability
```c
int msi_pos = pci_find_capability(dev(dev), PCI_CAP_ID_MSI);
```

### 读取 Message Control
```c
u16 ctrl;
pci_read_config_word(dev, msi_pos + PCI_MSI_FLAGS, &ctrl);

// 检查能力
bool is_64bit = (ctrl & PCI_MSI_FLAGS_64BIT) != 0;
bool has_masking = (ctrl & PCI_MSI_FLAGS_MASKBIT) != 0;
int mmc = (ctrl & PCI_MSI_FLAGS_QMASK) >> 1;
int mme = (ctrl & PCI_MSI_FLAGS_QSIZE) >> 4;
```

### 写入 MSI 地址和数据
```c
// 32 位地址
pci_write_config_dword(dev, msi_pos + PCI_MSI_ADDRESS_LO, address_lo);
pci_write_config_word(dev, msi_pos + PCI_MSI_DATA_32, data);

// 64 位地址
pci_write_config_dword(dev, msi_pos + PCI_MSI_ADDRESS_LO, address_lo);
pci_write_config_dword(dev, msi_pos + PCI_MSI_ADDRESS_HI, address_hi);
pci_write_config_word(dev, msi_pos + PCI_MSI_DATA_64, data);
```

### 启用 MSI
```c
u16 ctrl;
pci_read_config_word(dev, msi_pos + PCI_MSI_FLAGS, &ctrl);
ctrl |= PCI_MSI_FLAGS_ENABLE;
pci_write_config_word(dev, msi_pos + PCI_MSI_FLAGS, ctrl);
```

## MSI 与 INTx 的区别

| 特性 | INTx | MSI |
|------|------|-----|
| 信号线 | 专用线 | 内存写 |
| 共享 | 是 | 否 |
| 数量 | 1 | 最多 32 |
| 延迟 | 高 | 低 |
| 消息 | 无 | 有数据 |
| 屏蔽 | 共享 | 独立 |

## MSI 与 MSI-X 的区别

| 特性 | MSI | MSI-X |
|------|-----|-------|
| 最大向量数 | 32 | 2048 |
| 地址 | 固定 | 表驱动 |
| 数据 | 固定 | 表驱动 |
| 屏蔽 | 可选 | 每向量 |
| 掩序 | 简单 | 复杂 |

## 调试信息

```c
// 在 pci_msi_init() 中
if (!(ctrl & PCI_MSI_FLAGS_64BIT))
    dev->no_64bit_msi = 1;

// 在驱动中
dev_info(&pdev->dev, "Allocated %d MSI vectors\n", nvec);
```

## 错误处理

1. **MSI 不支持**:
   ```c
   if (!dev->msi_cap)
       return -EINVAL;
   ```

2. **分配失败**:
   ```c
   ret = pci_alloc_irq_vectors(pdev, 1, nvec, NULL);
   if (ret < 0) {
       dev_err(&pdev->ari->dev, "Failed to allocate MSI: %d\n", ret);
       return ret;
   }
   ```

3. **配置失败**:
   ```c
   if (pci_write_config_word(...) != PCIBIOS_SUCCESSFUL)
       return -EIO;
   ```

## 相关配置选项

- `CONFIG_PCI_MSI`: 启用 MSI 支持
- `CONFIG_GENERIC_MSI_IRQ_DOMAIN`: 通用 MSI IRQ domain
- `CONFIG_IRQ_REMAP`: IRQ 重映射支持

## 参考资料

- PCI Local Bus Specification 3.0+, Section 6.8
- PCIe Base Specification Revision 1.0+, Section 6.8.1
- Linux 内核源码: `drivers/pci/msi/`
- 头文件: `include/linux/msi.h`, `include/uapi/linux/pci_regs.h`
