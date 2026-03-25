# PCIe Capabilities 初始化总览

## 概述

本文档总结了 Linux 内核中 PCIe Capabilities 的初始化流程和软件实现。每个 capability 都按照 PCIe SPEC 定义和 Linux 内核实现进行详细分析。

## Capabilities 列表

| 序号 | Capability | ID | 类型 | 文档 |
|------|-----------|-----|------|------|
| 1 | Enhanced Allocation (EA) | 0x14 | 标准 | 01-Enhanced-Allocation-EA.md |
| 2 | MSI | 0x05 | 标准 | 02-MSI.md |
| 3 | MSI-X | 0x11 | 标准 | 03-MSI-X.md |
| 4 | Power Management (PM) | 0x01 | 标准 | 04-Power-Management-PM.md |
| 5 | Vital Product Data (VPD) | 0x03 | 标准 | 08-Vital-Product-Data-VPD.md |
| 6 | Advanced Error Reporting (AER) | 0x01 | 扩展 | 05-Advanced-Error-Reporting-AER.md |
| 7 | Downstream Port Containment (DPC) | 0x1D | 扩展 | 10-Downstream-Port-Containment-DPC.md |
| 8 | Single Root I/O Virtualization (SR-IOV) | 0x10 | 扩展 | 06-SR-IOV.md |
| 9 | Address Translation Services (ATS) | 0x0F | 扩展 | 09-Address-Translation-Services-ATS.md |
| 10 | Page Request Interface (PRI) | 0x13 | 扩展 | - |
| 11 | Process Address Space ID (PASID) | 0x1B | 扩展 | - |
| 12 | Precision Time Measurement (PTM) | 0x1F | 扩展 | 07-Precision-Time-Measurement-PTM.md |
| 13 | Access Control Services (ACS) | 0x0D | 扩展 | - |
| 14 | Root Complex Event Collector (RCEC) | 0x07 | 扩展 | - |
| 15 | Data Object Exchange (DOE) | 0x2E | 扩展 | - |
| 16 | TLP Processing Hints (TPH) | 0x17 | 扩展 | - |
| 17 | Resizable BAR (REBAR) | 0x15 | 扩展 | - |
| 18 | Device 3 Capabilities | 0x2F | 扩展 | - |
| 19 | Integrity and Data Encryption (IDE) | 0x30 | 扩展 | - |
| 20 | Native PCIe Enclosure Management (NPEM) | 0x29 | 扩展 | - |

## 初始化顺序

Capabilities 按照以下顺序在 `pci_init_capabilities()` 中初始化：

```c
void pci_init_capabilities(struct pci_dev *dev)
{
    pci_ea_init(dev);        // Enhanced Allocation
    pci_msi_init(dev);       // MSI
    pci_msix_init(dev);      // MSI-X
    
    pci_allocate_cap_save_buffers(dev);
    
    pci_imm_ready_init(dev);   // Immediate Readiness
    pci_pm_init(dev);        // Power Management
    pci_vpd_init(dev);       // Vital Product Data
    pci_configure_ari(dev);   // Alternative Routing-ID
    pci_iov_init(dev);       // SR-IOV
    pci_ats_init(dev);       // ATS
    pci_pri_init(dev);       // PRI
    pci_pasid_init(dev);     // PASID
    pci_acs_init(dev);       // ACS
    pci_ptm_init(dev);       // PTM
    pci_aer_init(dev);       // AER
    pci_dpc_init(dev);       // DPC
    pci_rcec_init(dev);      // RCEC
    pci_doe_init(dev);       // DOE
    pci_tph_init(dev);       // TPH
    pci_rebar_init(dev);     // REBAR
    pci_dev3_init(dev);      // Device 3
    pci_ide_init(dev);       // IDE
}
```

## Capability 查找机制

### 标准 Capability 查找

```c
int pci_find_capability(struct pci_dev *dev, u8 cap)
{
    u8 pos;
    u8 id;
    u8 ttl = PCI_FIND_CAP_TTL;
    int ent = PCI_CAPABILITY_LIST;
    
    pos = dev->hdr_type == PCI_HEADER_TYPE_CARDBUS ?
          PCI_CB_CAPABILITY_LIST : PCI_CAPABILITY_LIST;
    
    pci_read_config_byte(dev, pos, &pos);
    
    while (ttl--) {
        if (pos < PCI_STD_HEADER_SIZEOF)
            break;
        
        pos = ALIGN_DOWN(pos, 4);
        pci_read_config_byte(dev, pos + PCI_CAP_LIST_ID, &id);
        
        if (id == 0xff)
            break;
        
        if (id == cap)
            return pos;
        
        pci_read_config_byte(dev, pos + PCI_CAP_LIST_NEXT, &pos);
    }
    
    return 0;
}
```

### 扩展 Capability 查找

```c
int pci_find_ext_capability(struct pci_dev *dev, u16 cap)
{
    u16 pos;
    u16 id;
    u16 ttl = PCI_FIND_CAP_TTL;
    int ent = PCI_CFG_SPACE_SIZE;
    
    pos = PCI_CFG_SPACE_SIZE;
    
    while (ttl--) {
        if (pos < PCI_CFG_SPACE_SIZE)
            break;
        
        pos = ALIGN_DOWN(pos, 4);
        pci_read_config_word(dev, pos, &id);
        
        if (id == 0 || id == 0xffff)
            break;
        
        if (PCI_EXT_CAP_ID(id) == cap)
            return pos;
        
        pos = PCI_EXT_CAP_NEXT(id);
    }
    
    return 0;
}
```

## 配置空间布局

### 标准 Configuration Space
```
Offset  Size  Description
0x00    4    Device ID (Vendor + Device)
0x04    4    Command
0x08    4    Class Code + Revision
0x0C    4    Cache Line Size + Latency Timer
0x10    4    Header Type + BIST
0x14    4    BAR 0
0x18    4    BAR 1
0x1C    4    BAR 2
0x20    4    BAR 3
0x24    4    BAR 4
0x28    4    BAR 5
0x2C    4    CardBus CIS
0x30    4    Subsystem Vendor + Device ID
0x34    1    Capability Pointer
0x35    1    Interrupt Line
0x36    1    Interrupt Pin
0x37    1    Min_Gnt
0x38    1    Max_Lat
```

### 扩展 Configuration Space (PCIe)
```
Offset  Size  Description
0x100   4    Extended Capability Header (开始)
...      N    Extended Capabilities
0x1000  4K   扩展 Configuration Space 结束
```

## Root Port 特殊处理

Root Port 作为 PCIe 拓扑的根节点，需要特殊处理：

1. **Capability 初始化**: Root Port 通常不实现某些 capabilities（如 EA, VPD）
2. **错误报告**: Root Port 实现 AER 和 DPC，负责错误汇聚
3. **电源管理**: Root Port 支持电源管理，影响下游设备
4. **时间测量**: Root Port 可以作为 PTM Root

## 设备类型差异

### Endpoint 设备
- 支持大多数 capabilities
- 使用 MSI/MSI-X 进行中断
- 可以使用 ATS/PRI/PASID 进行虚拟化
- 支持 AER 错误报告

### Root Complex Integrated Endpoint
- 类似 Endpoint，但集成在 Root Complex 中
- 可能不支持某些外部可见的 capabilities

### Bridge 设备
- 支持 Bridge 特定的 capabilities
- 可能不支持某些 Endpoint capabilities
- 支持 PCIe Bridge capabilities

### Root Port
- 作为错误报告的汇聚点
- 支持 AER 和 DPC
- 可以作为 PTM Root

## 保存和恢复

### Capability 保存缓冲区

```c
// 分配保存缓冲区
pci_allocate_cap_save_buffers(dev);

// 保存 capability 状态
pci_save_state(dev);

// 恢复 capability 状态
pci_restore_state(dev);
```

### 保存/恢复特定 Capability

```c
// AER
pci_save_aer_state(dev);
pci_restore_aer_state(dev);

// PTM
pci_save_ptm_state(dev);
pci_restore_ptm_state(dev);

// SR-IOV
pci_restore_iov_state(dev);

// ATS
pci_restore_ats_state(dev);

// PRI
pci_restore_pri_state(dev);

// PASID
pci_restore_pasid_state(dev);

// DPC
pci_save_dpc_state(dev);
pci_restore_dpc_state(dev);
```

## 调试和调试

### Capability 调试信息

```c
// PCIe Capability
dev_info(dev, "PCIe capability enabled\n");

// MSI/MSI-X
dev_info(dev, "MSI enabled\n");
dev_info(dev, "MSI-X enabled, table size: %d\n", table_size);

// SR-IOV
dev_info(dev, "PF: Total VFs=%d, Initial VFs=%d\n",
         total_vfs, initial_vfs);

// AER
dev_info(dev, "AER capability enabled\n");

// PTM
dev_info(dev, "PTM enabled, granularity: %d ns\n", granularity);
```

### Debugfs 接口

```c
// PTM debugfs
struct pci_ptm_debugfs *ptm_debugfs = pcie_ptm_create_debugfs(&pdev->dev, pdev, &ptm_ops);

// 使用 debugfs
echo "1" > /sys/kernel/debug/pcie_ptm_xxxx/context_update
cat /sys/kernel/debug/pcie_ptm_xxxx/local_clock
```

## 性能考虑

1. **初始化顺序**: 某些 capabilities 依赖其他 capabilities 的初始化
2. **配置空间访问**: 减少配置空间访问次数
3. **保存/恢复状态**: 优化电源管理性能

4. **中断处理**: 使用 MSI/MSI-X 而非 INTx 提高性能

## 错误处理

### Capability 初始化失败

```c
// AER 初始化失败
if (!dev->aer_cap) {
    dev->aer_cap = 0;
    return;
}

// SR-IOV 初始化失败
ret = pci_iov_init(dev);
if (ret) {
    dev_err(dev, "SR-IOV init failed: %d\n", ret);
    return ret;
}
```

### Capability 启用失败

```c
// MSI 启用失败
ret = pci_enable_msi(dev, nvec);
if (ret) {
    dev_err(dev, "MSI enable failed: %d\n", ret);
    return ret;
}

// PTM 启用失败
ret = pci_enable_ptm(dev, &granularity);
if (ret) {
    dev_err(dev, "PTM enable failed: %d\n", ret);
    return ret;
}
```

## 相关配置选项

- `CONFIG_PCI`: 启用 PCI 支持
- `CONFIG_PCIEASPM`: ASPM 支持
- `CONFIG_PCIEAER`: AER 支持
- `CONFIG_PCIE_DPC`: DPC 支持
- `CONFIG_PCI_IOV`: SR-IOV 支持
- `CONFIG_PCI_ATS`: ATS 支持
- `CONFIG_PCI_PRI`: PRI 支持
- `CONFIG_PCI_PASID`: PASID 支持
- `CONFIG_PCIE_PTM`: PTM 支持

## 参考资料

- PCIe Base Specification
- Linux 内核源码: `drivers/pci/`
- 头文件: `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
