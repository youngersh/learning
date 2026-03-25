# Root Complex Event Collector (RCEC) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_RCEC (0x07)  
**规范参考**: PCIe RCEC Specification  
**功能**: 收集来自多个 RCiEPs 的错误事件

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x07)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    4    RCEC Capability
              - Bit 0: RCIEC Associated Endpoint Association Present
              - Bit 1: Bitmap Valid
              - Bit 2: RCEC Bus Number Valid
0x08    4    RCEC Associated Endpoint Bitmap
0x0C    4    RCEC Bus Number
              - Bits [7:0]: Next Bus Number
              - Bits [15:8]: Last Bus Number
```

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/pcie/rcec.c`  
**函数**: `pci_rcec_init()` (行 149-163)

```c
void pci_rcec_init(struct pci_dev *dev)
{
    int pos;
    u32 cap, header;

    // 1. 查找 RCEC capability
    pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_RCEC);
    if (!pos)
        return;

    // 2. 读取 RCEC Capability
    pci_read_config_dword(dev, pos + PCI_RCEC_CAP, &cap);

    // 3. 检查是否是 RCEC 设备
    if (pci_pcie_type(dev) != PCI_EXP_TYPE_RC_EC) {
        // 不是 RCEC 设备，可能是 RCiEP
        pcie_link_rcec(dev, rcec_add_ep, NULL);
        return;
    }

    // 4. 检查 Associated Endpoint Association
    if (cap & PCI_RCEC_CAP_AEP_ASSOC) {
        // RCEC 与 RCiEPs 关联
        pcie_link_rcec(dev, rcec_add_ep, NULL);
    }
}
```

### 添加 RCiEP

**函数**: `rcec_add_ep()` (内部函数)

```c
static void rcec_add_ep(struct pci_dev *rcec, struct pci_dev *ep)
{
    struct rcec_ea *ea;

    // 1. 分配 RCEC EA 结构
    ea = kzalloc(sizeof(*ea), GFP_KERNEL);
    if (!ea)
        return;

    // 2. 读取 Bus Number
    pci_read_config_dword(ep, ep->rcec_cap + PCI_RCEC_BUSN, &ea->bitmap);

    // 3. 添加到 RCEC 的 EA 列表
    pcie_link_rcec(rcec, rcec_add_ep, ea);
}
```

### 链接 RCiEP

**函数**: `pcie_link_rcec()` (行 78-101)

```c
void pcie_link_rcec(struct pci_dev *rcec,
                     int (*cb)(struct pci_dev *, void *),
                     void *userdata)
{
    struct pci_dev *pdev;
    int pos;
    u32 cap, header;
    u32 bitmap;

    // 1. 读取 RCEC Capability
    pos = pci_find_ext_capability(rcec, PCI_EXT_CAP_ID_RCEC);
    if (!pos)
        return;

    // 2. 读取 RCEC Capability
    pci_read_config_dword(rcec, pos + PCI_RCEC_CAP, &cap);

    // 3. 检查 Associated Endpoint Association
    if (!(cap & PCI_RCEC_CAP_AEP_ASSOC))
        return;

    // 4. 读取 Bitmap
    pci_read_config_dword(rcec, rcec->rcec_cap + PCI_RCEC_BITMAP, &bitmap);

    // 5. 遍历 Bitmap 中的每个位
    for (i = 0; i < 32; i++) {
        if (!(bitmap & (1 << i)))
            continue;

        // 6. 查找对应的 RCiEP
        pdev = pci_get_slot(rcec->bus, i);
        if (!pdev)
            continue;

        // 7. 调用回调处理 RCiEP
        cb(pdev, userdata);
    }
}
```

### 遍历 RCiEP

**函数**: `pcie_walk_rcec()` (行 789-808)

```c
void pcie_walk_rcec(struct pci_dev *rcec,
                    int (*cb)(struct pci_dev *, void *),
                    void *userdata)
{
    // 使用 pcie_link_rcec 遍历所有关联的 RCiEP
    pcie_link_rcec(rcec, cb, userdata);
}
```

### 退出 RCEC

**函数**: `pci_rcec_exit()` (行 165-171)

```c
void pci_rcec_exit(struct pci_dev *dev)
{
    // 清理 RCEC 相关资源
    // 当前实现为空函数
}
```

## Root Port 处理

Root Port 本身不实现 RCEC capability，因为它是 Bridge 设备。RCEC 是一个独立的 Endpoint 类型设备。

## 设备驱动使用

### 1. 检查 RCEC 支持
```c
// 检查是否是 RCEC 设备
if (pci_pcie_type(pdev) == PCI_EXP_TYPE_RC_EC) {
    dev_info(&pdev->dev, "RCEC device detected\n");
    
    // 读取 RCEC Capability
    int pos = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_RCEC);
    if (!pos)
        return;
    
    // 读取 RCEC Capability
    u32 cap;
    pci_read_config_dword(pdev, pos + PCI_RCEC_CAP, &cap);
    
    // 检查 Associated Endpoint Association
    if (cap & PCI_RCEC_CAP_AEP_ASSOC) {
        dev_info(&pdev->dev, "RCEC Associated Endpoint Association supported\n");
    }
}
```

### 2. 遍历关联的 RCiEPs

```c
// 遍历所有关联的 RCiEPs
pcie_walk_rcec(pdev, my_rcec_callback, NULL);

// RCiEP 回调处理
static int my_rcec_callback(struct pci_dev *rciep, void *data)
{
    // 处理 RCiEP
    dev_info(&rciep->dev, "Processing RCiEP\n");
    
    return 0;
}
```

## 调用流程

```
设备挂起:
  └─> pci_init_capabilities()
       └─> pci_rcec_init()
            ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_RCEC)
            ├─> pci_read_config_dword()  // 读取 RCEC Capability
            ├─> 检查设备类型
            └─> pcie_link_rcec()  // 链接 RCiEPs
                 └─> pci_read_config_dword()  // 读取 Bitmap
                 └─> 遍历位查找 RCiEP
```

RCEC 错误处理:
  └─> AER 检测到错误
       └─> 查找对应的 RCEC
       └─> pcie_link_rcec()  // 遍历关联的 RCiEPs
            └─> 通知 RCiEP
```

## 配置空间访问

### 查找 RCEC Capability
```c
int rcec_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_RCEC);
```

### 读取 RCEC Capability
```c
u32 cap;
pci_read_config_dword(dev, rcec_pos + PCI_RCEC_CAP, &cap);

// 检查 Associated Endpoint Association
bool aep_assoc = (cap & PCI_RCEC_CAP_AEP_ASSOC) != 0;

// 检查 Bitmap Valid
bool bitmap_valid = (cap & PCI_RCEC_CAP_BITMAP_VALID) != 0;

// 检查 Bus Number Valid
bool busn_valid = (cap & PCI_RCEC_CAP_BUSN_VALID) != 0;
```

### 读取 RCEC Bus Number
```c
u32 busn;
pci_read_config_dword(dev, rcec_pos + PCI_RCEC_BUSN, &busn);

// 获取 Next Bus Number
u8 next_busn = PCI_RCEC_BUSN_NEXT(busn);

// 获取 Last Bus Number
u8 last_busn = PCI_RCEC_BUSN_LAST(busn);
```

### 读取 RCEC Bitmap
```c
u32 bitmap;
pci_read_config_dword(dev, rcec_pos + PCI_RCEC_BITMAP, &bitmap);

// Bitmap 包含 32 位，每个位对应一个 RCiEP
// 位 1 表示对应的 RCiEP 存在
```

## RCEC 与 AER 的关系

```
AER 错误 → RCEC

当 AER 检测到错误时，通过 RCEC 查找对应的 RCiEP 并通知
```

## 性能考虑

1. **错误汇聚**: RCEC 集中错误报告到 RCEC
2. **Bitmap 遍历**: 需要遍历 32 位来查找 RCiEP
3. **关联管理**: 维护 RCEC 与 RCiEPs 的关联

## 调试信息

```c
// RCEC 初始化
dev_info(&pdev->dev, "RCEC device detected\n");

// Associated Endpoint Association
if (cap & PCI_RCEC_CAP_AEP_ASSOC) {
    dev_info(&pdev->dev, "RCEC Associated Endpoint Association supported\n");
}
```

## 错误处理

1. **RCEC 不支持**:
   ```c
   if (!pci_find_ext_capability(dev, PCI_EXT_CAP_ID_RCEC))
       return;
   ```

2. **Bitmap 无效**:
   ```c
   if (!(cap & PCI_RCEC_CAP_BITMAP_VALID))
       return;
   ```

3. **RCiEP 查找失败**:
   ```c
   pdev = pci_get_slot(rcec->bus, i);
   if (!pdev) {
       dev_err(rcec, "Failed to find RCiEP at slot %d\n", i);
       continue;
   }
   ```

## 相关配置选项

- `CONFIG_PCIEPORTBUS`: PCIe Port Bus 支持（RCEC 的依赖）
- `CONFIG_PCIEAER`: AER 支持（RCEC 的依赖）

## 参考资料

- PCIe Root Complex Event Collector Specification
- PCIe Base Specification Revision 5.0+, Section 7.8.4
- Linux 内核源码: `drivers/pci/pcie/rcec.c`
- 头文件: `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
