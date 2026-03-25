# Enhanced Allocation (EA) Capability 详细分析

## 概述

**Capability ID**: PCI_CAP_ID_EA (0x14)  
**规范参考**: PCIe r3.0+  
**功能**: 替代传统 BAR 的资源分配机制，提供更灵活的资源描述

## PCIe SPEC 定义

### Capability 结构
```
Offset  Size  Description
0x00    1    Capability ID (0x14)
0x01    1    Next Capability Pointer
0x02    2    Capability Header
0x04    1    Number of Entries (BEI count)
0x05-0x07 3    Reserved
0x08    N    First EA Entry (for Type 0)
0x0C    N    First EA Entry (for Type 1)
```

### EA Entry 结构
```
Offset  Size  Description
0x00    4    Entry Header
              - Bits [2:0]: Entry Size (ES)
              - Bits [7:4]: BAR Equivalent Indicator (BEI)
              - Bits [15:8]: Primary Properties (PP)
              - Bits [23:16]: Secondary Properties (SP)
              - Bit 31: Enable
0x04    4    Base Address (Lower 32 bits)
0x08    4    Max Offset
0x0C    4    Base Address (Upper 32 bits, optional)
```

### BEI (BAR Equivalent Indicator) 值
- 0-5: 对应 BAR 0-5
- 6: Bridge 资源
- 7: Equivalent Not Indicated
- 8: ROM
- 9-14: VF BAR 0-5
- 15: Reserved

### Primary Properties (PP) 值
- 0x00: Non-Prefetchable Memory
- 0x01: Prefetchable Memory
- 0x02: I/O Space
- 0x03: VF Prefetchable Memory
- 0x04: VF Non-Prefetchable Memory
- 0x05: Bridge Non-Prefetchable Memory
- 0x06: Bridge Prefetchable Memory
- 0x07: Bridge I/O Space

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/pci.c`  
**函数**: `pci_ea_init()` (行 3368-3394)

```c
void pci_ea_init(struct pci_dev *dev)
{
    int ea;
    u8 num_ent;
    int offset;
    int i;

    // 1. 查找 EA capability
    ea = pci_find_capability(dev, PCI_CAP_ID_EA);
    if (!ea)
        return;

    // 2. 读取 entry 数量
    pci_bus_read_config_byte(dev->bus, dev->devfn, ea + PCI_EA_NUM_ENT,
                    &num_ent);
    num_ent &= PCI_EA_NUM_ENT_MASK;

    offset = ea + PCI_EA_FIRST_ENT;

    // 3. Bridge 设备跳过 DWORD 2
    if (dev->hdr_type == PCI_HEADER_TYPE_BRIDGE)
        offset += 4;

    // 4. 解析每个 EA entry
    for (i = 0; i < num_ent; ++i)
        offset = pci_ea_read(dev, offset);
}
```

### EA Entry 解析

**函数**: `pci_ea_read()` (行 3244-3364)

```c
static int pci_ea_read(struct pci_dev *dev, int offset)
{
    struct resource *res;
    const char *res_name;
    int ent_size, ent_offset = offset;
    resource_size_t start, end;
    unsigned long flags;
    u32 dw0, bei, base, max_offset;
    u8 prop;
    bool support_64 = (sizeof(resource_size_t) >= 8);

    // 1. 读取 Entry Header
    pci_read_config_dword(dev, ent_offset, &dw0);
    ent_offset += 4;

    // 2. 计算 entry 大小
    ent_size = (FIELD_GET(PCI_EA_ES, dw0) + 1) << 2;

    // 3. 检查是否启用
    if (!(dw0 & PCI_EA_ENABLE))
        goto out;

    // 4. 提取 BEI 和 Properties
    bei = FIELD_GET(PCI_EA_BEI, dw0);
    prop = FIELD_GET(PCI_EA_PP, dw0);

    // 5. 处理保留属性
    if (prop > PCI_EA_P_BRIDGE_IO && prop < PCI_EA_P_MEM_RESERVED)
        prop = FIELD_GET(PCI_EA_SP, dw0);

    // 6. 获取对应的 resource
    res = pci_ea_get_resource(dev, bei, prop);
    res_name = pci_resource_name(dev, bei);
    if (!res) {
        pci_err(dev, "Unsupported EA entry BEI: %u\n", bei);
        goto out;
    }

    // 7. 转换为 resource flags
    flags = pci_ea_flags(dev, prop);

    // 8. 读取 Base Address
    pci_read_config_dword(dev, ent_offset, &base);
    start = (base & PCI_EA_FIELD_MASK);
    ent_offset += 4;

    // 9. 读取 Max Offset
    pci_read_config_dword(dev, ent_offset, &max_offset);
    ent_offset += 4;

    // 10. 处理 64 位地址
    if (base & PCI_EA_IS_64) {
        u32 base_upper;
        pci_read_config_dword(dev, ent_offset, &base_upper);
        ent_offset += 4;
        flags |= IORESOURCE_MEM_64;
        if (support_64)
            start |= ((u64)base_upper << 32);
    }

    // 11. 计算结束地址
    end = start + (max_offset | 0x03);

    // 12. 设置 resource
    res->start = start;
    res->end = end;
    res->flags = flags;
    res->name = res_name;

    pci_info(dev, "BEI %d %pR: from Enhanced Allocation, properties %#02x\n",
           bei, res, prop);

out:
    return offset + ent_size;
}
```

### Resource 获取

**函数**: `pci_ea_get_resource()` (行 3226-3241)

```c
static struct resource *pci_ea_get_resource(struct pci_dev *dev, u8 bei,
                       u8 prop)
{
    // 标准 BAR 0-5
    if (bei <= PCI_EA_BEI_BAR5 && prop <= PCI_EA_P_IO)
        return &dev->resource[bei];
    
    // VF BAR 0-5
    else if (bei >= PCI_EA_BEI_VF_BAR0 && bei <= PCI_EA_BEI_VF_BAR5 &&
         (prop == PCI_EA_P_VF_MEM || prop == PCI_EA_P_VF_MEM_PREFETCH))
        return &dev->resource[PCI_IOV_RESOURCES +
                      bei - PCI_EA_BEI_VF_BAR0];
    
    // ROM
    else if (bei == PCI_EA_BEI_ROM)
        return &dev->resource[PCI_ROM_RESOURCE];
    
    else
        return NULL;
}
```

### Flags 转换

**函数**: `pci_ea_flags()` (行 3203-3224)

```c
static unsigned long pci_ea_flags(struct pci_dev *dev, u8 prop)
{
    unsigned long flags = IORESOURCE_PCI_FIXED | IORESOURCE_PCI_EA_BEI;

    switch (prop) {
    case PCI_EA_P_MEM:
    case PCI_EA_P_VF_MEM:
        flags |= IORESOURCE_MEM;
        break;
    case PCI_EA_P_MEM_PREFETCH:
    case PCI_EA_P_VF_MEM_PREFETCH:
        flags |= IORESOURCE_MEM | IORESOURCE_PREFETCH;
        break;
    case PCI_EA_P_IO:
        flags |= IORESOURCE_IO;
        break;
    default:
        return 0;
    }

    return flags;
}
```

## Root Port 处理

Root Port 通常不实现 EA capability，因为它是 Bridge 设备。如果实现，EA entry 的 BEI 会设置为 6 (Bridge 资源)，用于描述桥接器后的资源窗口。

## 设备驱动使用

### 1. 检查 EA 支持
```c
if (pci_find_capability(pdev, PCI_CAP_ID_EA)) {
    // 设备支持 EA
}
```

### 2. 读取 EA 资源
EA 资源在 `pci_ea_init()` 中已经解析并填充到 `dev->resource[]`，驱动可以直接使用标准的 PCI resource API：

```c
// 获取 BAR 0
struct resource *res = &pdev->resource[0];

// 检查是否来自 EA
if (res->flags & IORESOURCE_PCI_EA_BEI) {
    // 这是 EA 分配的资源
}

// 获取资源信息
resource_size_t start = pci_resource_start(pdev, 0);
resource_size_t len = pci_resource_len(pdev, 0);
```

### 3. 映射资源
```c
void __iomem *base;

// 检查资源类型
if (resource_type(res) == IORESOURCE_MEM) {
    base = ioremap(pci_resource_start(pdev, bar),
                   pci_resource_len(pdev, bar));
} else if (resource_type(res) == IORESOURCE_IO) {
    base = ioport_map(pci_resource_start(pdev, bar),
                     pci_resource_len(pdev, bar));
}
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_ea_init()
                 ├─> pci_find_capability(PCI_CAP_ID_EA)
                 ├─> pci_bus_read_config_byte()  // 读取 num_ent
                 └─> pci_ea_read()  // 解析每个 entry
                       ├─> pci_read_config_dword()  // 读取 header
                       ├─> pci_ea_get_resource()  // 获取 resource
                       ├─> pci_ea_flags()  // 转换 flags
                       ├─> pci_read_config_dword()  // 读取 base
                       ├─> pci_read_config_dword()  // 读取 max_offset
                       └─> 设置 dev->resource[]
```

## 配置空间访问

### 查找 EA Capability
```c
int ea_pos = pci_find_capability(dev, PCI_CAP_ID_EA);
```

### 读取 EA Header
```c
u8 num_ent;
pci_read_config_byte(dev, ea_pos + PCI_EA_NUM_ENT, &num_ent);
num_ent &= PCI_EA_NUM_ENT_MASK;
```

### 读取 EA Entry
```c
u32 header;
pci_read_config_dword(dev, entry_offset, &header);

u32 base;
pci_read_config_dword(dev, entry_offset + 4, &base);

u32 max_offset;
pci_read_config_dword(dev, entry_offset + 8, &max_offset);
```

## 特殊处理

### Bridge 设备
Bridge 设备的 EA entry 从 offset + 12 开始（跳过 DWORD 2），因为 DWORD 2 包含固定的 Secondary 和 Subordinate bus number。

### 64 位地址
如果 Base Address 字段的 bit 0 设置，表示 64 位地址，需要读取额外的 DWORD 作为高 32 位。

### VF 资源
VF 的 BEI 范围是 9-14，对应 VF BAR 0-5。这些资源存储在 `dev->resource[PCI_IOV_RESOURCES + n]`。

## 与传统 BAR 的区别

| 特性 | 传统 BAR | EA |
|------|---------|-----|
| 资源描述 | 硬编码在 BAR 中 | 显式描述 |
| 灵活性 | 固定格式 | 可扩展属性 |
| VF 支持 | 需要单独 capability | 统一描述 |
| Bridge 资源 | 需要特殊处理 | 显式 BEI=6 |
| 对齐 | 由 BAR 大小决定 | 由 MaxOffset 决定 |

## 调试信息

```c
pci_info(dev, "BEI %d %pR: from Enhanced Allocation, properties %#02x\n",
         bei, res, prop);
```

## 错误处理

1. **不支持 BEI**:
   ```c
   pci_err(dev, "Unsupported EA entry BEI: %u\n", bei);
   ```

2. **不支持属性**:
   ```c
   pci_err(dev, "Unsupported EA properties: %#x\n", prop);
   ```

## 相关配置选项

- `CONFIG_PCI`: 启用 PCI 支持
- `CONFIG_PCI_IOV`: 启用 SR-IOV 支持（用于 VF 资源）

## 参考资料

- PCIe Base Specification Revision 3.0+, Section 7.8
- Linux 内ra源码: `drivers/pci/pci.c`
- 头文件: `include/uapi/linux/pci_regs.h`
