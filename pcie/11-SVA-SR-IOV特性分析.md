# SR-IOV (Single Root I/O Virtualization) 特性分析

## 概述

SR-IOV 是 PCIe Base Specification 中定义的 I/O 虚拟化技术，允许一个 PCIe 设备（物理函数 PF）创建多个虚拟设备（虚拟函数 VF），每个 VF 可以被分配给不同的虚拟机或进程，实现硬件资源的共享。

**关键特性**：
- 单个物理功能 (PF) 可创建最多 256 个虚拟功能 (VF)
- VF 共享 PF 的硬件资源（BARs、中断等）
- 支持动态启用/禁用 VF
- 与 Resizable BAR 扩展能力集成

**实现文件**：
- `drivers/pci/iov.c` - SR-IOV 核心实现
- `drivers/pci/rebar.c` - Resizable BAR 支持
- `include/linux/pci.h` - 核心数据结构
- `include/uapi/linux/pci_regs.h` - PCIe 寄存器定义

## PCIe SPEC 要求参考

### SR-IOV Extended Capability 结构

根据 PCIe Base Spec 9.x 章节，SR-IOV 扩展能力包含以下关键寄存器：

| 偏移 | 寄存器 | 说明 | SPEC 位置 |
|-------|--------|------|-----------|
| 0x04 | SR-IOV Capabilities | 能力标志（VFM等） | SR-IOV sec 3.3.1 |
| 0x08 | SR-IOV Control | 控制（VF Enable、Memory Space Enable等） | SR-IOV sec 3.3.2 |
| 0x0A | SR-IOV Status | 状态（VF Migration等） | SR-IOV sec 3.3.3 |
| 0x0C | Initial VFs | 初始 VF 数量 | SR-IOV sec 3.3.4 |
| 0x0E | Total VFs | 总 VF 数量 | SR-IOV sec 3.3.5 |
| 0x10 | Num VFs | 当前启用的 VF 数量 | SR-IOV sec 3.3.6 |
| 0x12 | Function Dependency Link | 功能依赖链接 | SR-IOV sec 3.3.7 |
| 0x14 | First VF Offset | 第一个 VF 的路由 ID 偏移 | SR-IOV sec 3.3.8 |
| 0x16 | Following VF Stride | VF 之间的步长 | SR-IOV sec 3.3.9 |
| 0x1A | VF Device ID | VF 的设备 ID | SR-IOV sec 3.3.10 |
| 0x1C | Supported Page Sizes | 支持的页大小 | SR-IOV sec 3.3.11 |
| 0x20 | System Page Size | 系统页大小 | SR-IOV sec 3.3.12 |
| 0x24 | VF BAR[0-5] | VF 的 BAR 寄存器 | SR-IOV sec 3.3.13 |

**关键 SPEC 要求**：

1. **First VF Offset 和 VF Stride 可能随 NumVFs 变化** (SR-IOV sec 3.3.10/3.3.11)
   - Linux 通过 `pci_iov_set_numvfs()` 动态读取这些值

2. **VF BARs 大小必须页对齐** (SR-IOV sec 3.3.11)
   - 系统页大小由 Supported Page Sizes 位掩码指定
   - BAR 大小必须是 2^n 字节，n >= PAGE_SHIFT

3. **VF 内存空间必须先禁用后修改** (SR-IOV sec 3.3.2)
   - 修改 VF BAR 前必须清除 VF Enable 和 Memory Space Enable

4. **VF 的 Bus/Devfn 计算** (SR-IOV sec 3.3.8/3.3.9)
   - Bus Number = PF Bus Number + (PF Devfn + Offset + Stride * VF_ID) >> 8
   - Devfn = (PF Devfn + Offset + Stride * VF_ID) & 0xFF

## 核心数据结构

### struct pci_sriov (drivers/pci/pci.h:566)

```c
struct pci_sriov {
    int     pos;            /* SR-IOV Capability位置 */
    int     nres;           /* 资源数量 (启用的VF BAR数量) */
    u32     cap;            /* SR-IOV Capabilities (PCI_SRIOV_CAP_VFM等) */
    u16     ctrl;           /* SR-IOV Control */
    u16     total_VFs;       /* 硬件支持的总VF数 */
    u16     initial_VFs;     /* 初始VF数 */
    u16     num_VFs;         /* 当前启用的VF数 */
    u16     offset;          /* First VF Routing ID offset */
    u16     stride;          /* Following VF stride */
    u16     vf_device;       /* VF Device ID */
    u32     pgsz;           /* Page size for BAR alignment */
    u8      link;           /* Function Dependency Link */
    u8      max_VF_buses;    /* Max buses consumed by VFs */
    u16     driver_max_VFs;  /* Driver支持的最大VF数 */
    struct pci_dev *dev;     /* 最低编号的PF */
    struct pci_dev *self;    /* 当前PF */
    u32     class;          /* VF Device Class/Revision */
    u8      hdr_type;       /* VF Header Type */
    u16     subsystem_vendor; /* VF Subsystem Vendor */
    u16     subsystem_device; /* VF Subsystem Device */
    resource_size_t barsz[PCI_SRIOV_NUM_BARS];  /* VF BAR大小 */
    u16     vf_rebar_cap;    /* VF Resizable BAR能力位置 */
    bool    drivers_autoprobe; /* VF驱动自动探测 */
};
```

### struct pci_dev 中的 SR-IOV 相关字段 (include/linux/pci.h)

```c
struct pci_dev {
    // ... 其他字段 ...

#ifdef CONFIG_PCI_ATS
    union {
        struct pci_sriov *sriov;    /* PF: SR-IOV info */
        struct pci_dev *physfn;   /* VF: 相关的PF */
    };
#endif

    unsigned int is_physfn:1;  /* 标识是否为PF */
    unsigned int is_virtfn:1;  /* 标识是否为VF */
};
```

**关键说明**：
- PF 设备：`is_physfn=1`, `sriov` 指针有效
- VF 设备：`is_virtfn=1`, `physfn` 指针指向关联的 PF

## SR-IOV 初始化流程

### 流程图
```
设备探测 (pci_setup_device)
    ↓
发现 SR-IOV Extended Capability (pci_find_ext_capability)
    ↓
pci_iov_init() → sriov_init()
    ↓
1. 读取 SR-IOV Capability 寄存器
   - Total VFs, VF Device ID, Capabilities
   - Initial VFs, Supported Page Sizes
    ↓
2. 计算系统页大小
   - pgsz &= ~(pgsz - 1)  (保留最高设置位)
   - 写入 System Page Size 寄存器
    ↓
3. 探测 VF BARs 大小
   - __pci_size_stdbars() 读取 VF BAR0-5
   - 检查页对齐
   - 设置 resource 大小 = VF_BAR_SIZE * Total_VFs
    ↓
4. 初始化 pci_sriov 结构
   - 保存 TotalVFs, InitialVFs, VF_DID
   - 保存 barsz[] (每个VF BAR的大小)
    ↓
5. 查找 VF Resizable BAR Capability
   - vf_rebar_cap = pci_find_ext_capability(PCI_EXT_CAP_ID_VF_REBAR)
    ↓
6. 计算最大 VF 总线数量
   - compute_max_vf_buses() 迭代所有可能的NumVFs
   - 验证 offset 和 stride 有效性
   - 计算最后的VF的bus number
    ↓
7. 设置 dev->is_physfn = 1
```

### 关键函数实现

#### sriov_init() (iov.c:797)

```c
static int sriov_init(struct pci_dev *dev, int pos)
{
    // 1. 禁用 VF (如果已启用)
    pci_read_config_word(dev, pos + PCI_SRIOV_CTRL, &ctrl);
    if (ctrl & PCI_SRIOV_CTRL_VFE) {
        pci_write_config_word(dev, pos + PCI_SRIOV_CTRL, 0);
        ssleep(1);
    }

    // 2. 查找是否有其他 PF (用于 VF 管理)
    list_for_each_entry(pdev, &dev->bus->devices, bus_list)
        if (pdev->is_physfn)
            goto found;
    // 设置 ARI 标志 (如果支持)
    if (pci_ari_enabled(dev->bus))
        ctrl |= PCI_SRIOV_CTRL_ARI;
found:
    pci_write_config_word(dev, pos + PCI_SRIOV_CTRL, ctrl);

    // 3. 读取 Total VFs
    pci_read_config_word(dev, pos + PCI_SRIOV_TOTAL_VF, &total);
    if (!total)
        return 0;  // 不支持 SR-IOV

    // 4. 计算系统页大小
    pci_read_config_dword(dev, pos + PCI_SRIOV_SUP_PGSIZE, &pgsz);
    i = PAGE_SHIFT > 12 ? PAGE_SHIFT - 12 : 0;
    pgsz &= ~((1 << i) - 1);  // 保留比内核页大的支持位
    if (!pgsz)
        return -EIO;

    pgsz &= ~(pgsz - 1);  // 只保留最高设置位
    pci_write_config_dword(dev, pos + PCI_SRIOV_SYS_PGSIZE, pgsz);

    // 5. 探测 VF BARs
    __pci_size_stdbars(dev, PCI_SRIOV_NUM_BARS,
                       pos + PCI_SRIOV_BAR, sriovbars);

    // 6. 初始化每个 VF BAR
    for (i = 0; i < PCI_SRIOV_NUM_BARS; i++) {
        bar64 = __pci_read_base(dev, pci_bar_unknown, res,
                                pos + PCI_SRIOV_BAR + i * 4,
                                &sriovbars[i]);
        if (!res->flags)
            continue;

        // 检查页对齐
        if (resource_size(res) & (PAGE_SIZE - 1)) {
            rc = -EIO;
            goto failed;
        }

        iov->barsz[i] = resource_size(res);
        // 设置 PF resource 大小为所有 VF 的总和
        resource_set_size(res, resource_size(res) * total);
        i += bar64;
        nres++;
    }

    // 7. 保存其他信息
    iov->pos = pos;
    iov->nres = nres;
    iov->ctrl = ctrl;
    iov->total_VFs = total;
    iov->driver_max_VFs = total;
    pci_read_config_word(dev, pos + PCI_SRIOV_VF_DID, &iov->vf_device);
    iov->pgsz = pgsz;
    pci_read_config_dword(dev, pos + PCI_SRIOV_CAP, &iov->cap);
    pci_read_config_byte(dev, pos + PCI_SRIOV_FUNC_LINK, &iov->link);

    // 8. 查找 VF Resizable BAR Capability
    iov->vf_rebar_cap = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_VF_REBAR);

    // 9. 计算最大总线数
    dev->is_physfn = 1;
    rc = compute_max_vf_buses(dev);
}
```

## VF 创建和枚举流程

### 流程图
```
用户写入 sysfs: sriov_numvfs = N
    ↓
sriov_numvfs_store()
    ↓
检查 PF 驱动是否提供 sriov_configure 回调
    ↓
调用 PF 驱动的 sriov_configure(dev, N)
    ↓
PF 驱动调用 pci_enable_sriov(dev, N)
    ↓
sriov_enable(dev, N)
    ↓
1. 参数验证
   - N <= total_VFs
   - N <= initial_VFs (如果支持 VFM)
    ↓
2. 资源检查
   - 遍历所有 VF BAR
   - 检查: VF_BAR_SIZE * N <= PF_RESOURCE_SIZE
   - 启用 PF 资源 (iowrite_enable())
    ↓
3. 总线范围检查
   -最后 VF 的 bus number <= busn_res.end
    ↓
4. 调用 pcibios_sriov_enable() (平台特定)
    ↓
5. 启用 VF
   - 设置 NumVFs = N
   - 读取更新后的 offset 和 stride
   - 设置 SR-IOV Control: VFE=1, MSE=1
    ↓
6. 创建 VF 设备
   - sriov_add_vfs(dev, initial_VFs)
   - 循环创建每个 VF
    ↓
7. 为每个 VF:
   pci_iov_add_virtfn(dev, vf_id)
   ├─ 1. 创建/查找 PCI 总线
   │    virtfn_add_bus()
   ├─ 2. 分配 pci_dev 结构
   │    pci_iov_scan_device()
   │    ├─ 设置 devfn
   │    ├─ 设置 vendor/device/vf_device
   │    ├─ 设置 is_virtfn = 1
   │    ├─ 设置 physfn (指向PF)
   │    └─ pci_setup_device(virtfn)
   ├─ 3. 分配 VF BAR 资源
   │    for (i = 0; i < PCI_SRIOV_NUM_BARS; i++)
   │        virtfn->resource[i].start = pf_res->start + vf_barsz * vf_id
   │        virtfn->resource[i].end = start + vf_barsz
   │        request_resource(pf_res, &virtfn->resource[i])
   ├─ 4. 创建 sysfs 链接
   │    pci_iov_sysfs_link()
   │    ├─ /sys/.../pf/virtfn0 -> vf
   │    └─ /sys/.../vf/physfn -> pf
   └─ 5. 注册 VF 设备
        pci_bus_add_device(virtfn)
```

### VF Bus/Devfn 计算实现

#### pci_iov_virtfn_bus() (iov.c:24)

```c
int pci_iov_virtfn_bus(struct pci_dev *dev, int vf_id)
{
    if (!dev->is_physfn)
        return -EINVAL;

    // Bus Number = PF_Bus + (PF_Devfn + Offset + Stride * VF_ID) >> 8
    return dev->bus->number + ((dev->devfn + dev->sriov->offset +
                        dev->sriov->stride * vf_id) >> 8);
}
```

#### pci_iov_virtfn_devfn() (iov.c:32)

```c
int pci_iov_virtfn_devfn(struct pci_dev *dev, int vf_id)
{
    if (!dev->is_physfn)
        return -EINVAL;

    // Devfn = (PF_Devfn + Offset + Stride * VF_ID) & 0xFF
    return (dev->devfn + dev->sriov->offset +
            dev->sriov->stride * vf_id) & 0xff;
}
```

**示例计算**：
```
假设:
  PF Bus=10, PF Devfn=0x10 (Slot: 2, Func: 0)
  offset=0x20, stride=0x08

VF0:
  routing_id = 0x10 + 0x20 + 0x08*0 = 0x30
  bus = 10 + (0x30 >> 8) = 10 + 0 = 10
  devfn = 0x30 & 0xFF = 0x30 (Slot: 6, Func: 0)

VF1:
  routing_id = 0x10 + 0x20 + 0x08*1 = 0x38
  bus = 10 + (0x38 >> 8) = 10 + 0 = 10
  devfn = 0x38 & 0xFF = 0x38 (Slot: 7, Func: 0)

VF32 (stride=0x08时跨总线):
  routing_id = 0x10 + 0x20 + 0x08*32 = 0x10 + 0x20 + 0x100 = 0x130
  bus = 10 + (0x130 >> 8) = 10 + 1 = 11
  devfn = 0x130 & 0xFF = 0x30 (Slot: 6, Func: 0)
```

### pci_iov_add_virtfn() 实现 (iov.c:346)

```c
int pci_iov_add_virtfn(struct pci_dev *dev, int id)
{
    // 1. 创建/查找总线
    bus = virtfn_add_bus(dev->bus, pci_iov_virtfn_bus(dev, id));

    // 2. 扫描设备
    virtfn = pci_iov_scan_device(dev, id, bus);
    //    - 分配 pci_dev
    //    - 设置 devfn, vendor, device (vf_device)
    //    - 设置 is_virtfn=1, physfn=dev
    //    - pci_setup_device()

    // 3. 设置 VF BAR 资源
    for (i = 0; i < PCI_SRIOV_NUM_BARS; i++) {
        idx = pci_resource_num_from_vf_bar(i);
        res = &dev->resource[idx];  // PF 的资源
        if (!res->parent)
            continue;

        size = pci_iov_resource_size(dev, idx);  // 单个 VF 的 BAR 大小
        // VF BAR 的起始地址 = PF BAR 起始 + VF_ID * VF_BAR_SIZE
        resource_set_range(&virtfn->resource[i],
                       res->start + size * id, size);
        // 从 PF 资源中分配子资源
        request_resource(res, &virtfn->resource[i]);
    }

    // 4. 添加设备到系统
    pci_device_add(virtfn, virtfn->bus);

    // 5. 创建 sysfs 链接
    pci_iov_sysfs_link(dev, virtfn, id);

    // 6. 触发 VF 驱动探测
    pci_bus_add_device(virtfn);
}
```

## Resizable BAR 与 SR-IOV 集成

### PCIe Resizable BAR Extended Capability

根据 PCIe Base Spec 7.8.6 章节：

| 寄存器 | 偏移 | 说明 |
|--------|-------|------|
| Capabilities | +0x00 | 可支持的大小位掩码 (bit n = 2^n MB) |
| Control | +0x04 | BAR Index + BAR Size + Resizable |

**大小编码**：
- 0 = 1MB
- 1 = 2MB
- ...
- 31 = 128TB

### VF Resizable BAR 支持

Linux 支持两种 Resizable BAR：
1. **PF Resizable BAR** - `PCI_EXT_CAP_ID_REBAR`
2. **VF Resizable BAR** - `PCI_EXT_CAP_ID_VF_REBAR`

#### 大小转换函数 (rebar.c)

```c
// 字节 → size 编码
int pci_rebar_bytes_to_size(u64 bytes)
{
    bytes = roundup_pow_of_two(bytes);
    return ilog2(bytes) - ilog2(SZ_1M);  // log2(bytes) - 20
}

// size 编码 → 字节
resource_size_t pci_rebar_size_to_bytes(int size)
{
    return 1ULL << (size + ilog2(SZ_1M));  // 2^(size+20)
}
```

#### 查询支持的大小 (rebar.c:108)

```c
u64 pci_rebar_get_possible_sizes(struct pci_dev *pdev, int bar)
{
    pos = pci_rebar_find_pos(pdev, bar);
    if (pos < 0)
        return 0;

    pci_read_config_dword(pdev, pos + PCI_REBAR_CAP, &cap);
    cap = FIELD_GET(PCI_REBAR_CAP_SIZES, cap);

    return cap;  // 返回支持的大小位掩码
}
```

**示例**：
```
cap = 0x00000FF3  → 支持的大小：1MB, 2MB, 4MB, 8MB, 16MB, 32MB, 64MB, 128MB
```

### VF BAR Resize 实现

#### pci_iov_vf_bar_set_size() (iov.c:1340)

```c
int pci_iov_vf_bar_set_size(struct pci_dev *dev, int resno, int size)
{
    // 1. 检查是否为 IOV 资源
    if (!pci_resource_is_iov(resno))
        return -EINVAL;

    // 2. 检查 VF 内存空间是否已启用
    if (pci_iov_is_memory_decoding_enabled(dev))
        return -EBUSY;

    // 3. 检查大小是否支持
    if (!pci_rebar_size_supported(dev, resno, size))
        return -EINVAL;

    // 4. 设置新的 BAR 大小
    return pci_rebar_set_size(dev, resno, size);
}
```

#### pci_rebar_set_size() (rebar.c:204)

```c
int pci_rebar_set_size(struct pci_dev *pdev, int bar, int size)
{
    pos = pci_rebar_find_pos(pdev, bar);

    // 1. 读取 Control 寄存器
    pci_read_config_dword(pdev, pos + PCI_REBAR_CTRL, &ctrl);

    // 2. 清除旧大小，设置新大小
    ctrl &= ~PCI_REBAR_CTRL_BAR_SIZE;
    ctrl |= FIELD_PREP(PCI_REBAR_CTRL_BAR_SIZE, size);

    // 3. 写回 Control 寄存器
    pci_write_config_dword(pdev, pos + PCI_REBAR_CTRL, ctrl);

    // 4. 如果是 VF BAR，更新缓存
    if (pci_resource_is_iov(bar))
        pci_iov_resource_set_size(pdev, bar, size);

    return 0;
}
```

### 状态恢复

#### sriov_restore_vf_rebar_state() (iov.c:932)

```c
static void sriov_restore_vf_rebar_state(struct pci_dev *dev)
{
    pos = pci_iov_vf_rebar_cap(dev);
    if (!pos)
        return;

    // 读取支持的可调整 BAR 数量
    pci_read_config_dword(dev, pos + PCI_VF_REBAR_CTRL, &ctrl);
    nbars = FIELD_GET(PCI_VF_REBAR_CTRL_NBAR_MASK, ctrl);

    // 恢复每个 BAR 的大小
    for (i = 0; i < nbars; i++, pos += 8) {
        pci_read_config_dword(dev, pos + PCI_VF_REBAR_CTRL, &ctrl);
        bar_idx = FIELD_GET(PCI_VF_REBAR_CTRL_BAR_IDX, ctrl);

        // 将字节大小转换为 size 编码
        size = pci_rebar_bytes_to_size(dev->sriov->barsz[bar_idx]);

        ctrl &= ~PCI_VF_REBAR_CTRL_BAR_SIZE;
        ctrl |= FIELD_PREP(PCI_VF_REBAR_CTRL_BAR_SIZE, size);
        pci_write_config_dword(dev, pos + PCI_VF_REBAR_CTRL, ctrl);
    }
}
```

## Sysfs 控制接口

### PF 设备的 sysfs 属性

| 文件 | 读写 | 说明 |
|------|------|------|
| sriov_totalvfs | RO | 硬件支持的总VF数 |
| sriov_numvfs | RW | 当前启用的VF数 |
| sriov_offset | RO | First VF Offset |
| sriov_stride | RO | VF Stride |
| sriov_vf_device | RO | VF Device ID |
| sriov_drivers_autoprobe | RW | VF驱动自动探测 |
| sriov_vf_total_msix | RO | VF MSI-X 总向量数 (CONFIG_PCI_MSI) |

### VF 设备的 sysfs 属性

| 文件 | 读写 | 说明 |
|------|------|------|
| sriov_vf_msix_count | WO | 设置 VF MSI-X 向量数 (CONFIG_PCI_MSI) |

### Sysfs 链接

启用 VF 后，系统会创建双向 sysfs 链接：

```
/sys/bus/pci/devices/0000:10:02.0/sriov_numvfs  (PF)
/sys/bus/pci/devices/0000:10:06.0/virtfn0          (PF -> VF0 链接)
/sys/bus/pci/devices/0000:10:06.0/physfn           (VF -> PF 链接)
/sys/bus/pci/devices/0000:10:38.0/virtfn1          (PF -> VF1 链接)
```

### sriov_numvfs_store() 实现 (iov.c:463)

```c
static ssize_t sriov_numvfs_store(struct device *dev,
                                  struct device_attribute *attr,
                                  const char *buf,最终 size_t count)
{
    pdev = to_pci_dev(dev);

    // 1. 解析参数
    if (kstrtou16(buf, 0, &num_vfs) < 0)
        return -EINVAL;

    // 2. 检查范围
    if (num_vfs > pci_sriov_get_totalvfs(pdev))
        return -ERANGE;

    device_lock(&pdev->dev);

    // 3. 如果数量相同，直接返回
    if (num_vfs == pdev->sriov->num_VFs)
        goto exit;

    // 4. 检查 PF 驱动
    if (!pdev->driver) {
        pci_info(pdev, "no driver bound to device; cannot configure SR-IOV\n");
        ret = -ENOENT;
        goto exit;
    }

    if (!pdev->driver->sriov_configure) {
        pci_info(pdev, "driver does not support SR-IOV configuration via sysfs\n");
        ret = -ENOENT;
        goto exit;
    }

    // 5. 禁用 VFs (num_vfs == 0)
    if (num_vfs == 0) {
        ret = pdev->driver->sriov_configure(pdev, 0);
        goto exit;
    }

    // 6. 检查是否已启用 VFs
    if (pdev->sriov->num_VFs) {
        pci_warn(pdev, "%d VFs already enabled. Disable before enabling %d VFs\n",
                 pdev->sriov->num_VFs, num_vfs);
        ret = -EBUSY;
        goto exit;
    }

    // 7. 启用 VFs
    ret = pdev->driver->sriov_configure(pdev, num_vfs);
    if (ret < 0)
        goto exit;

    if (ret != num_vfs)
        pci_warn(pdev, "%d VFs requested; only %d enabled\n",
                 num_vfs, ret);

exit:
    device_unlock(&pdev->dev);
    return ret ? : count;
}
```

## 资源管理

### VF BAR 资源分配策略

Linux 为 VF BAR 资源分配采用 **子资源分配** 模式：

```
PF Resource (BAR0):
  总大小 = VF_BAR_SIZE * Total_VFs
  start: 0x10000000
  end:   0x10000000 + VF_BAR_SIZE * 256 - 1

VF0 BAR0:
  parent: PF Resource (BAR0)
  start: 0x10000000 + 0 * VF_BAR_SIZE
  end:   0x10000000 + 1 * VF_BAR_SIZE - 1

VF1 BAR0:
  parent: PF Resource (BAR0)
  start: 0x10000000 + 1 * VF_BAR_SIZE
  end:   0x10000000 + 2 * VF_BAR_SIZE - 1

...
```

**关键点**：
- PF 的 resource 大小 = VF_BAR_SIZE * Total_VFs
- 每个 VF 的 resource 是 PF resource 的子资源
- 使用 `request_resource()` 进行分配
- VF 的 BAR 大小存储在 `dev->sriov->barsz[]`

### pci_update_resource() 路径 (setup-res.c:126)

```c
void pci_update_resource(struct pci_dev *dev, int resno)
{
    if (resno <= PCI_ROM_RESOURCE)
        pci_std_update_resource(dev, resno);  // 标准 BAR
    else if (pci_resource_is_iov(resno))
        pci_iov_update_resource(dev, resno);  // VF BAR
}
```

### PCI 标准 BAR 更新 (setup-res.c:25)

```c
static void pci_std_update_resource(struct pci_dev *dev, int resno)
{
    // 1. VF 的 BARs 是只读零 (SR-IOV spec 3.4.1.11)
    if (dev->is_virtfn)
        return;

    // 2. 检查资源有效性
    if (!res->flags || res->flags & IORESOURCE_UNSET)
        return;

    // 3. 转换到总线地址
    pcibios_resource_to_bus(dev->bus, &region, res);
    new = region.start;

    // 4. 64 位 BAR 需要特殊处理
    disable = (res->flags & IORESOURCE_MEM_64) && !dev->mmio_always_on;
    if (disable) {
        // 禁用内存解码以避免半更新 BAR 冲突
        pci_read_config_word(dev, PCI_COMMAND, &cmd);
        pci_write_config_word(dev, PCI_COMMAND, cmd & ~PCI_COMMAND_MEMORY);
    }

    // 5. 写低位 dword
    pci_write_config_dword(dev, reg, new);
    pci_read_config_dword(dev, reg, &check);

    // 6. 写高位 dword (64位 BAR)
    if (res->flags & IORESOURCE_MEM_64) {
        new = region.start >> 32;
        pci_write_config_dword(dev, reg + 4, new);
        pci_read_config_dword(dev, reg + 4, &check);
    }

    // 7. 恢复内存解码
    if (disable)
        pci_write_config_word(dev, PCI_COMMAND, cmd);
}
```

## 配置空间访问寄存器

### SR-IOV 寄存器偏移定义 (include/uapi/linux/pci_regs.h:972)

```c
#define PCI_SRIOV_CAP         0x04   /* SR-IOV Capabilities */
#define  PCI_SRIOV_CAP_VFM    0x00000001  /* VF Migration Capable */
#define  PCI_SRIOV_CAP_INTR(x) ((x) >> 21)  /* Interrupt Message Number */

#define PCI_SRIOV_CTRL        0x08   /* SR-IOV Control */
#define  PCI_SRIOV_CTRL_VFE   0x0001  /* VF Enable */
#define  PCI_SRIOV_CTRL_VFM   0x0002  /* VF Migration Enable */
#define  PCI_SRIOV_CTRL_INTR  0x0004  /* VF Migration Interrupt Enable */
#define  PCI_SRIOV_CTRL_MSE   0x0008  /* VF Memory Space Enable */
#define  PCI_SRIOV_CTRL_ARI   0x0010  /* ARI Capable Hierarchy */

#define PCI_SRIOV_STATUS      0x0a   /* SR-IOV Status */
#define  PCI_SRIOV_STATUS_VFM  0x0001  /* VF Migration Status */

#define PCI_SRIOV_INITIAL_VF  0x0c   /* Initial VFs */
#define PCI_SRIOV_TOTAL_VF    0x0e   /* Total VFs */
#define PCI_SRIOV_NUM_VF      0x10   /* Number of VFs */
#define PCI_SRIOV_FUNC_LINK  0x12   /* Function Dependency Link */
#define PCI_SRIOV_VF_OFFSET   0x14   /* First VF Offset */
#define PCI_SRIOV_VF_STRIDE  0x16   /* Following VF Stride */
#define PCI_SRIOV_VF_DID     0x1a   /* VF Device ID */
#define PCI_SRIOV_SUP_PGSIZE 0x1c   /* Supported Page Sizes */
#define PCI_SRIOV_SYS_PGSIZE 0x20   /* System Page Size */
#define PCI_SRIOV_BAR        0x24   /* VF BAR0 */
#define PCI_SRIOV_NUM_BARS   6       /* Number of VF BARs */
```

### Resizable BAR 寄存器偏移

```c
#define PCI_REBAR_CAP        0x00   /* Capabilities */
#define  PCI_REBAR_CTRL       0x04   /* Control */
#define  PCI_REBAR_CTRL_BAR_IDX_MASK 0x00000007  /* BAR Index */
#define  PCI_REBAR_CTRL_BAR_SIZE   0x0000001F  /* BAR Size */
#define  PCI_REBAR_CTRL_NBAR_MASK  0x00E00000  /* Number of BARs */
```

## 关键 API 总结

### 初始化和释放

| API | 位置 | 说明 |
|-----|------|------|
| pci_iov_init() | iov.c | 初始化 SR-IOV capability |
| pci_iov_release() | iov.c | 释放 SR-IOV 资源 |
| pci_iov_remove() | iov.c | PF 驱动移除后清理状态 |

### VF 管理

| API | 位置 | 说明 |
|-----|------|------|
| pci_enable_sriov() | iov.c | 启用指定数量的 VF |
| pci_disable_sriov() | iov.c | 禁用所有 VF |
| pci_num_vf() | iov.c | 返回当前 VF 数量 |
| pci_sriov_set_totalvfs() | iov.c | 设置驱动支持的最大 VF 数 |
| pci_sriov_get_totalvfs() | iov.c | 获取支持的最大 VF 数 |
| pci_sriov_configure_simple() | iov.c | 简单 SR-IOV 配置助手 |

### VF 地址计算

| API | 位置 | 说明 |
|-----|------|------|
| pci_iov_virtfn_bus() | iov.c | 计算 VF 的 bus number |
| pci_iov_virtfn_devfn() | iov.c | 计算 VF 的 devfn |
| pci_iov_vf_id() | iov.c | 从 VF 设备获取 VF ID |

### 资源管理

| API | 位置 | 说明 |
|-----|------|------|
| pci_iov_resource_size() | iov.c | 获取 VF BAR 大小 |
| pci_iov_resource_set_size() | iov.c | 设置 VF BAR 大小 |
| pci_iov_update_resource() | iov.c | 更新 VF BAR 到硬件 |

### Resizable BAR

| API | 位置 | 说明 |
|-----|------|------|
| pci_rebar_bytes_to_size() | rebar.c | 字节转 size 编码 |
| pci_rebar_size_to_bytes() | rebar.c | size 编码转字节 |
| pci_rebar_get_possible_sizes() | rebar.c | 获取支持的大小掩码 |
| pci_rebar_size_supported() | rebar.c | 检查大小是否支持 |
| pci_rebar_get_max_size() | rebar.c | 获取最大支持大小 |
| pci_rebar_get_current_size() | rebar.c | 获取当前大小 |
| pci_rebar_set_size() | rebar.c | 设置 BAR 大小 |
| pci_resize_resource() | rebar.c | 完整的资源调整流程 |

### VF 特定 Resizable BAR

| API | 位置 | 说明 |
|-----|------|------|
| pci_iov_vf_bar_set_size() | iov.c | 设置 VF BAR 大小 |
| pci_iov_vf_bar_get_sizes() | iov.c | 获取可用的 VF BAR 大小 |

## 驱动使用示例

### PF 驱动启用 SR-IOV

```c
static int my_pf_probe(struct pci_dev *pdev)
{
    int num_vfs = 32;  // 要启用的 VF 数量

    // 1. 检查设备是否支持 SR-IOV
    if (!pci_is_pcie(pdev) || !pdev->is_physfn)
        return -ENODEV;

    // 2. 设置驱动支持的最大 VF 数
    pci_sriov_set_totalvfs(pdev, num_vfs);

    // 3. 启用设备
    pci_enable_device(pdev);

    return 0;
}

static void my_pf_remove(struct pci_dev *pdev)
{
    // 1. 禁用 SR-IOV (如果在 probe 中启用了)
    if (pdev->is_physfn && pci_sriov_get_totalvfs(pdev) > 0)
        pci_disable_sriov(pdev);

    pci_disable_device(pdev);
}

static int my_pf_sriov_configure(struct pci_dev *pdev, int num_vfs)
{
    if (num_vfs == 0) {
        // 禁用所有 VF
        pci_disable_sriov(pdev);
        return 0;
    }

    // 启用指定数量的 VF
    return pci_enable_sriov(pdev, num_vfs);
}

static struct pci_driver my_pf_driver = {
    .name = "my_pf_driver",
    .id_table = my_pf_id_table,
    .probe = my_pf_probe,
    .remove = my_pf_remove,
    .sriov_configure = my_pf_sriov_configure,
};
```

### VF 驱动访问 PF 数据

```c
static int my_vf_probe(struct pci_dev *pdev)
{
    struct pci_dev *pf;
    struct my_pf_data *pf_data;

    // 1. 检查是否为 VF
    if (!pdev->is_virtfn)
        return -ENODEV;

    // 2. 获取关联的 PF
    pf = pci_physfn(pdev);

    // 3. 获取 PF 的驱动数据
    pf_data = pci_iov_get_pf_drvdata(pdev, &my_pf_driver);
    if (IS_ERR(pf_data))
        return PTR_ERR(pf_data);

    // 4. 使用 PF 数据...
    return 0;
}
```

### Resizable VF BAR 使用

```c
static int my_pf_setup_vf_bars(struct pci_dev *pdev)
{
    u32 sizes;
    int bar_idx, new_size;

    // 1. 查询 BAR0 支持的大小
    bar_idx = pci_resource_num_from_vf_bar(0);
    sizes = pci_rebar_get_possible_sizes(pdev, bar_idx);

    // 2. 检查是否支持 8MB (size=3)
    if (BIT(3) & sizes) {
        // 3. 设置 VF BAR0 大小为 8MB
        new_size = 3;  // 3 = 8MB
        pci_iov_vf_bar_set_size(pdev, bar_idx, new_size);
    }

    return 0;
}
```

## 配置空间保护

### VF Enable 和 Memory Space Enable

SR-IOV spec 要求在修改 VF BARs 前必须禁用 VF：

```c
// 启用 VFs (sriov_enable)
pci_cfg_access_lock(dev);  // 锁定配置空间访问
iov->ctrl |= PCI_SRIOV_CTRL_VFE | PCI_SRIOV_CTRL_MSE;
pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL, iov->ctrl);
msleep(100);  // 等待硬件处理
pci_cfg_access_unlock(dev);

// 禁用 VFs (sriov_disable)
iov->ctrl &= ~(PCI_SRIOV_CTRL_VFE | PCI_SRIOV_CTRL_MSE);
pci_cfg_access_lock(dev);
pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL, iov->ctrl);
ssleep(1);  // 等待 VF 完全禁用
pci_cfg_cfg_access_unlock(dev);
```

### 配置空间锁定机制

Linux 使用 `pci_cfg_access_lock/unlock()` 保护配置空间访问：

```c
void pci_cfg_access_lock(struct pci_dev *dev)
{
    raw_spin_lock_irq(&pci_lock);
    if (dev->block_cfg_access)
        pci_wait_cfg(dev);  // 等待访问被允许
    dev->block_cfg_access = 1;
    raw_spin_unlock_irq(&pci_lock);
}

void pci_cfg_access_unlock(struct pci_dev *dev)
{
    raw_spin_lock_irqsave(&pci_lock, flags);
    WARN_ON(!dev->block_cfg_access);
    dev->block_cfg_access = 0;
    raw_spin_unlock_irqrestore(&pci_lock, flags);
    wake_up_all(&pci_cfg_wait);  // 唤醒等待的访问者
}
```

**用途**：
- 防止在 BIST (Built-In Self Test) 期间访问配置空间
- 保护 D-state 转换期间的配置空间
- 防止并发修改配置空间

## 状态保存和恢复

### sriov_restore_state() (iov.c:956)

```c
static void sriov_restore_state(struct pci_dev *dev)
{
    struct pci_sriov *iov = dev->sriov;

    // 1. 检查 VF 是否已启用
    pci_read_config_word(dev, iov->pos + PCI_SRIOV_CTRL, &ctrl);
    if (ctrl & PCI_SRIOV_CTRL_VFE)
        return;

    // 2. 恢复 ARI 标志 (必须在 set_numvfs 之前)
    ctrl &= ~PCI_SRIOV_CTRL_ARI;
    ctrl |= iov->ctrl & PCI_SRIOV_CTRL_ARI;
    pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL, ctrl);

    // 3. 恢复 VF BARs
    for (i = 0; i < PCI_SRIOV_NUM_BARS; i++)
        pci_update_resource(dev, pci_resource_num_from_vf_bar(i));

    // 4. 恢复系统页大小
    pci_write_config_dword(dev, iov->pos + PCI_SRIOV_SYS_PGSIZE, iov->pgsz);

    // 5. 恢复 NumVFs (会读取 offset 和 stride)
    pci_iov_set_numvfs(dev, iov->num_VFs);

    // 6. 恢复 Control 寄存器
    pci_write_config_word(dev, iov->pos + PCI_SRIOV_CTRL, iov->ctrl);

    // 7. 如果 VF Enable，等待硬件处理
    if (iov->ctrl & PCI_SRIOV_CTRL_VFE)
        msleep(100);
}
```

### pci_restore_iov_state() (iov.c:1113)

```c
void pci_restore_iov_state(struct pci_dev *dev)
{
    if (dev->is_physfn) {
        // 1. 恢复 VF Resizable BAR 状态
        sriov_restore_vf_rebar_state(dev);

        // 2. 恢复 SR-IOV 状态
        sriov_restore_state(dev);
    }
}
```

## 总结

### SR-IOV 关键实现要点

1. **PF 和 VF 的关系**
   - PF 设备有 `is_physfn=1` 和 `sriov` 结构
   - VF 设备有 `is_virtfn=1` 和 `physfn` 指针
   - VF BAR 资源是 PF BAR 资源的子资源

2. **VF 地址计算**
   - Bus Number = PF Bus + (PF Devfn + Offset + Stride * VF_ID) >> 8
   - Devfn = (PF Devfn + Offset + Stride * VF_ID) & 0xFF
   - Offset 和 Stride 会随 NumVFs 变化，需动态读取

3. **资源分配策略**
   - PF 资源大小 = VF_BAR_SIZE * Total_VFs
   - VF 资源从 PF 资源中分配子资源
   - VF 起始地址 = PF 起始 + VF_ID * VF_BAR_SIZE

4. **Resizable BAR 集成**
   - 支持标准 Resizable BAR 和 VF Resizable BAR
   - 大小编码：0=1MB, 1=2MB, ..., 31=128TB
   - VF BAR 大小修改需先禁用 VF Memory Space

5. **配置空间保护**
   - 使用 `pci_cfg_access_lock/unlock()` 保护关键操作
   - VF Enable Bar 修改需要特殊保护
   - 64 位 BAR 需要暂时禁用内存解码

6. **驱动模型**
   - PF 驱动实现 `sriov_configure()` 回调
   - VF 驱动通过 `pci_iov_get_pf_drvdata()` 访问 PF 数据
   - VF 驱动通过 `pci_physfn()` 获取关联 PF

### PCIe SPEC 合规性检查清单

- [x] VF BAR 大小必须页对齐 (PAGE_SIZE)
- [x] NumVFs 变化时动态读取 Offset 和 Stride
- [x] VF BAR 修改前禁用 VF Memory Space
- [x] VF Enable 和 VF Migration 交互正确处理
- [x] 总线范围不超过可用总线号限制
- [x] VF BAR 资源不超过 PF 资源范围
- [x] Initial VFs <= Total VFs
- [x] 支持 VF Migration 时检查 VFM 能力
- [x] 配置空间访问正确锁定和解锁
- [x] 64 位 BAR 更新使用安全序列

### 相关文件清单

| 文件 | 功能 |
|------|------|
| drivers/pci/iov.c | SR-IOV 核心实现 |
| drivers/pci/rebar.c | Resizable BAR 实现 |
| drivers/pci/setup-res.c | 资源更新和分配 |
| drivers/pci/access.c | 配置空间访问保护 |
| include/linux/pci.h | 栓心数据结构 |
| include/linux/pci-ats.h | ATS 相关定义 |
| include/uapi/linux/pci.h | 用户空间 API |
| include/uapi/linux/pci_regs.h | PCIe 寄存器定义 |
