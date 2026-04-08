# PCIe Endpoint (端点) 框架分析

## 概述

`drivers/pci/endpoint` 目录实现了 Linux PCIe Endpoint 框架，这是一个让 SoC（如 ARM SoC）作为 PCIe Endpoint（而非传统的 Root Complex）连接到其他 PCIe 设备的框架。

**关键概念**：
- 传统 PCIe 拓扑：CPU/SoC 作为 Root Complex (RC)，外接 PCIe Endpoint (EP)
- PCIe Endpoint 模式：SoC 作为 Endpoint，可以连接到其他 PCIe Root Complex
- 典型应用：SoC 作为网络/存储/加速器卡连接到 x86 主机

**目录结构**：
```
drivers/pci/endpoint/
├── pci-epc-core.c          # Endpoint Controller (EPC) 核心库
├── pci-epf-core.c          # Endpoint Function (EPF) 核心库
├── pci-epc-mem.c           # EPC 地址空间管理
├── pci-ep-cfs.c            # ConfigFS 接口配置
├── pci-ep-msi.c            # MSI Doorbell 支持
└── functions/              # Endpoint Function 驱动
    ├── pci-epf-test.c      # 测试功能驱动
    ├── pci-epf-ntb.c       # NTB (Non-Transparent Bridge) - EP 控制器间桥接
    ├── pci-epf-vntb.c      # Virtual NTB - RC↔EP 桥接
    └── pci-epf-mhi.c       # MHI (Modem Host Interface)
```

## 核心架构

### 三层架构

```
┌─────────────────────────────────────────────────────┐
│                     应用程序层                                 │
└─────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────┐
│              Endpoint Function 驱动层 (pci-epf-*.c)          │
│                                                               │
│   • pci-epf-test  - 测试驱动                                    │
│   • pci-epf-ntb   - EP 控制器间 NTB 桥接                    │
│   • pci-epf-vntb  - Virtual NTB (RC↔EP 桥接)                   │
│   • pci-epf-mhi   - MHI (Modem Host Interface)                     │
└─────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────┐
│               EPF 核心库 (pci-epf-core.c)                  │
│                                                               │
│   • pci_epf_bind/unbind   # 绑定/解绑函数驱动                  │
│   • pci_epf_add_vepf      # 添加虚拟函数                          │
│   • pci_epf_add_epc       # 关联 EPC 和 EPF                        │
└─────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────┐
│              EPC 核心库 (pci-epc-core.c)                  │
│                                                               │
│   • pci_epc_get/put       # 获取/释放 EPC 设备                   │
│   • pci_epc_get_features  # 获取 EPC 能力                        │
│   • pci_epc_mem_alloc     # 分配 EPC 地址空间                    │
└─────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────┐
│            SoC PCIe Controller 驱动 (平台特定)                   │
│                                                               │
│   例：dw-pcie-ep.c (DesignWare PCIe Endpoint)                  │
│       cdns-pcie-ep.c (Cadence PCIe Endpoint)                      │
└─────────────────────────────────────────────────────┘
```

## 核心数据结构

### 1. struct pci_epc - Endpoint Controller (include/linux/pci-epc.h:169)

```c
struct pci_epc {
    struct device          dev;              /* EPC 设备 */
    struct list_head       pci_epf;          /* 此 EPC 中的 EPF 列表 */
    struct mutex          list_lock;         /* 保护 pci_epf 列表 */
    const struct pci_epc_ops *ops;          /* EPC 操作回调 */
    struct pci_epc_mem    **windows;         /* 地址窗口数组 */
    struct pci_epc_mem    *mem;              /* 默认地址窗口 */
    unsigned int          num_windows;       /* 支持的窗口数量 */
    u8                   max_functions;     /* 最大函数数 */
    u8                   *max_vfs;          /* 每个 PF 的最大 VF 数 */
    struct config_group   *group;            /* ConfigFS 组 */
    struct mutex          lock;              /* 保护 EPC ops */
    unsigned long          function_num_map;   /* 函数号位图 */
    int                   domain_nr;         /* PCI 域编号 */
    bool                  init_complete;     /* 初始化完成标志 */
};
```

**说明**：
- `dev` - EPC 是一个 Linux 设备，有 /sys/devices/pci_epc_xxx 入口
- `pci_epf` - 关联到此 EPC 的所有 Endpoint Function 列表
- `windows` - EPC 支持的多个地址窗口（用于不同的内存区域）
- `max_functions` - 此 EPC 支持的最大物理函数数量
- `max_vfs` - 每个物理函数支持的最大虚拟函数数

### 2. struct pci_epc_ops - EPC 操作 (include/linux/pci-epc.h:89)

```c
struct pci_epc_ops {
    int  (*write_header)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                       struct pci_epf_header *hdr);
    int  (*set_bar)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                   struct pci_epf_bar *epf_bar);
    void (*clear_bar)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                     struct pci_epf_bar *epf_bar);
    u64  (*align_addr)(struct pci_epc *epc, u64 pci_addr, size_t *size,
                       size_t *offset);
    int  (*map_addr)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                    phys_addr_t addr, u64 pci_addr, size_t size);
    void (*unmap_addr)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                      phys_addr_t addr);
    int  (*set_msi)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                   u8 nr_irqs);
    int  (*get_msi)(struct pci_epc *epc, u8 func_no, u8 vfunc_no);
    int  (*set_msix)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                    u16 nr_irqs, enum pci_barno, u32 offset);
    int  (*get_msix)(struct pci_epc *epc, u8 func_no, u8 vfunc_no);
    int  (*raise_irq)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                     unsigned int type, u16 interrupt_num);
    int  (*map_msi_irq)(struct pci_epc *epc, u8 func_no, u8 vfunc_no,
                       phys_addr_t phys_addr, u8 interrupt_num,
                       u32 entry_size, u32 *msi_data,
                       u32 *msi_addr_offset);
    int  (*start)(struct pci_epc *epc);
    void (*stop)(struct pci_epc *epc);
    const struct pci_epc_features* (*get_features)(struct pci_epc *epc,
                                              u8 func_no, u8 vfunc_no);
    struct module *owner;
};
```

**关键操作说明**：
- `write_header` - 写入 PCIe 配置空间头（VendorID, DeviceID 等）
- `set_bar/clear_bar` - 配置/清除 BAR（Base Address Register）
- `map_addr/unmap_addr` - 映射 CPU 物理地址到 PCIe 地址
- `set_msi/set_msix` - 配置 MSI/MSI-X 中断能力
- `raise_irq` - 向 RC 触发中断
- `start/stop` - 启动/停止 PCIe 链路
- `get_features` - 获取 EPC 支持的能力

### 3. struct pci_epf - Endpoint Function (include/linux/pci-epf.h:172)

```c
struct pci_epf {
    struct device          dev;              /* EPF 设备 */
    const char           *name;             /* EPF 名称 */
    struct pci_epf_header  *header;           /* 配置头 */
    struct pci_epf_bar   bar[PCI_STD_NUM_BARS];  /* BAR 信息 */
    u8                   msi_interrupts;    /* MSI 中断数 */
    u16                  msix_interrupts;   /* MSI-X 中断数 */
    u8                   func_no;           /* 物理函数号 */
    u8                   vfunc_no;          /* 虚拟函数号 */

    struct pci_epc       *epc;              /* 关联的 EPC */
    struct pci_epf       *epf_pf;           /* 父物理函数 (VF 用) */
    struct pci_epf_driver *driver;           /* EPF 驱动 */
    const struct pci_epf_device_id *id;       /* 设备 ID */

    struct list_head       list;              /* EPC 的 EPF 列表节点 */
    struct mutex          lock;              /* 保护 EPF ops */

    /* 支持 Secondary EPC (双控制器) */
    struct pci_epc       *sec_epc;          /* 次要 EPC */
    struct list_head       sec_epc_list;      /* 次要 EPF 列表 */
    struct pci_epf_bar   sec_epc_bar[PCI_STD_NUM_BARS];  /* 次要 BAR */
    u8                   sec_epc_func_no;   /* 次要函数号 */

    struct config_group   *group;            /* ConfigFS 组 */
    unsigned int          is_bound;          /* 已绑定标志 */
    unsigned int          is_vf;             /* 是否为虚拟函数 */
    unsigned long          vfunction_num_map;  /* VF 号位图 */
    struct list_head       pci_vepf;          /* 关联的 VF 列表 */
    const struct pci_epc_event_ops *event_ops;  /* 事件回调 */
    struct pci_epf_doorbell_msg *db_msg;      /* Doorbell 消息 */
    u16                  num_db;            /* Doorbell 数量 */
};
```

**说明**：
- `bar[]` - 存储 6 个标准 BAR 的映射信息
- `func_no` - 物理函数号 (0, 1, 2, ...)
- `vfunc_no` - 虚拟函数号 (SR-IOV VF 编号)
- `epc` - 关联的 EPC（可以是主 EPC 或次 EPC）
- `epf_pf` - 对于 VF，指向父物理函数
- `is_vf` - 标识此 EPF 为虚拟函数
- `event_ops` - 链路状态、Bus Master Enable 等事件回调
- `sec_epc` - 次要 EPC（用于双控制器架构）

### 4. struct pci_epf_ops - EPF 操作 (include/linux/pci-epf.h:65)

```c
struct pci_epf_ops {
    int  (*bind)(struct pci_epf *epf);
    void (*unbind)(struct pci_epf *epf);
    struct config_group *(*add_cfs)(struct pci_epf *epf,
                                    struct config_group *group);
};
```

- `bind` - EPF 驱动绑定到 EPC 时调用
- `unbind` - EPF 与 EPC 解绑时调用
- `add_cfs` - 添加功能特定的 ConfigFS 属性

### 5. struct pci_epf_bar - BAR 信息 (include/linux/pci-epf.h:123)

```c
struct pci_epf_bar {
    dma_addr_t    phys_addr;     /* 要映射到 BAR 的物理地址 */
    void          *addr;          /* 对应的虚拟地址 */
    size_t        size;           /* 地址空间大小 */
    size_t        mem_size;       /* 实际分配大小 (对齐后) */
    enum pci_barno barno;          /* BAR 编号 */
    int           flags;          /* BAR 标志 */
};
```

### 6. struct pci_epf_header - 配置头 (include/linux/pci-epf.h:45)

```c
struct pci_epf_header {
    u16  vendorid;        /* 设备制造商 ID */
    u16  deviceid;        /* 设备 ID */
    u8   revid;           /* 版本号 */
    u8   progif_code;     /* 编程接口代码 */
    u8   subclass_code;    /* 子类代码 */
    u8   baseclass_code;   /* 基类代码 */
    u8   cache_line_size;  /* 缓存行大小 */
    u16  subsys_vendor_id; /* 子系统供应商 ID */
    u16  subsys_id;        /* 子系统 ID */
    enum pci_interrupt_pin interrupt_pin; /* 中断引脚 */
};
```

### 7. struct pci_epc_mem - EPC 内存管理 (include/linux/pci-epc.h:140)

```c
struct pci_epc_mem {
    struct pci_epc_mem_window  window;   /* 地址窗口 */
    unsigned long             *bitmap;   /* 地址空间位图 */
    int                      pages;     /* 页数 */
    struct mutex              lock;      /* 保护位图 */
};

struct pci_epc_mem_window {
    phys_addr_t    phys_base;   /* 物理基址 */
    size_t        size;         /* 窗口大小 */
    size_t        page_size;    /* 页大小 */
};
```

## EPC 核心库 (pci-epc-core.c)

### 获取和释放 EPC

#### pci_epc_get() - 获取 EPC 设备 (pci-epc-core.c:51)

```c
struct pci_epc *pci_epc_get(const char *epc_name)
{
    // 1. 通过设备名称查找 EPC 设备
    dev = class_find_device_by_name(&pci_epc_class, epc_name);
    if (!dev)
        goto err;

    epc = to_pci_epc(dev);

    // 2. 增加 EPC 驱动模块引用计数
    if (try_module_get(epc->ops->owner))
        return epc;

err:
    put_device(dev);
    return ERR_PTR(ret);
}
```

**用途**：EPF 驱动通过名称获取 EPC 设备引用

### 获取 EPC 能力

#### pci_epc_get_features() - 获取 EPC 支持的能力 (pci-epc-core.c:139)

```c
const struct pci_epc_features *pci_epc_get_features(struct pci_epc *epc,
                                               u8 func_no, u8 vfunc_no)
{
    const struct pci_epc_features *epc_features;

    // 1. 验证函数号和虚拟函数号
    if (!pci_epc_function_is_valid(epc, func_no, vfunc_no))
        return NULL;

    // 2. 检查 EPC 是否支持 get_features
    if (!epc->ops->get_features)
        return NULL;

    mutex_lock(&epc->lock);

    // 3. 调用 EPC 驱动的 get_features
    epc_features = epc->ops->get_features(epc, func_no, vfunc_no);

    mutex_unlock(&epc->lock);
    return epc_features;
}
```

### 查找可用 BAR

#### pci_epc_get_next_free_bar() - 查找下一个可用 BAR (pci-epc-core.c:93)

```c
enum pci_barno pci_epc_get_next_free_bar(const struct pci_epc_features *epc_features,
                                       enum pci_barno bar)
{
    // 1. 如果前一个 BAR 是 64 位的，跳到下一个 BAR
    if (bar > 0 && epc_features->bar[bar - 1].only_64bit)
        bar++;

    // 2. 从指定 BAR 开始查找非保留 BAR
    for (i = bar; i < PCI_STD_NUM_BARS; i++) {
        if (epc_features->bar[i].type != BAR_RESERVED)
            return i;
    }

    return NO_BAR;
}
```

**用途**：EPF 驱动查找可用于配置的 BAR

## EPF 核心库 (pci-epf-core.c)

### EPF 绑定和解绑

#### pci_epf_bind() - 绑定 EPF 驱动 (pci-epf-core.c:59)

```c
int pci_epf_bind(struct pci_epf *epf)
{
    // 1. 检查 EPF 是否有驱动
    if (!epf->driver) {
        dev_WARN(dev, "epf device not bound to driver\n");
        return -EINVAL;
    }

    // 2. 增加 EPF 驱动模块引用计数
    if (!try_module_get(epf->driver->owner))
        return -EAGAIN;

    mutex_lock(&epf->lock);

    // 3. 绑定所有关联的虚拟函数
    list_for_each_entry(epf_vf, &epf->pci_vepf, list) {
        vfunc_no = epf_vf->vfunc_no;

        // 验证虚拟函数号
        if (vfunc_no < 1) {
            dev_err(dev, "Invalid virtual function number\n");
            ret = -EINVAL;
            goto ret;
        }

        // 检查 EPC 是否支持虚拟函数
        epc = epf->epc;
        func_no = epf->func_no;
        if (!IS_ERR_OR_NULL(epc)) {
            if (!epc->max_vfs) {
                dev_err(dev, "No support for virt function\n");
                ret = -EINVAL;
                goto ret;
            }

            if (vfunc_no > epc->max_vfs[func_no]) {
                dev_err(dev, "PF%d: Exceeds max vfunc number\n",
                         func_no);
                ret = -EINVAL;
                goto ret;
            }
        }

        // 设置 EPF 的函数号和 EPC
        epf_vf->func_no = epf->func_no;
        epf_vf->sec_epc_func_no = epf->sec_epc_func_no;
        epf_vf->epc = epf->epc;
        epf_vf->sec_epc = epf->sec_epc;

        // 调用 VF 驱动的 bind
        ret = epf_vf->driver->ops->bind(epf_vf);
        if (ret)
            goto ret;
        epf_vf->is_bound = true;
    }

    // 4. 绑定物理函数
    ret = epf->driver->ops->bind(epf);
    if (ret)
        goto ret;
    epf->is_bound = true;

    mutex_unlock(&epf->lock);
    return 0;

ret:
    mutex_unlock(&epf->lock);
    pci_epf_unbind(epf);  // 绑定失败，解绑
    return ret;
}
```

#### pci_epf_unbind() - 解绑 EPF 驱动 (pci-epf-core.c:31)

```c
void pci_epf_unbind(struct pci_epf *epf)
{
    struct pci_epf *epf_vf;

    if (!epf->driver) {
        dev_WARN(&epf->dev, "epf device not bound to driver\n");
        return;
    }

    mutex_lock(&epf->lock);

    // 1. 解绑所有关联的虚拟函数
    list_for_each_entry(epf_vf, &epf->pci_vepf, list) {
        if (epf_vf->is_bound)
            epf_vf->driver->ops->unbind(epf_vf);
    }

    // 2. 解绑物理函数
    if (epf->is_bound)
        epf->driver->ops->unbind(epf);

    mutex_unlock(&epf->lock);
    module_put(epf->driver->owner);
}
```

### 添加虚拟函数

#### pci_epf_add_vepf() - 添加虚拟 EPF (pci-epf-core.c:145)

```c
struct pci_epf *pci_epf_add_vepf(struct pci_epf *epf_pf,
                                struct pci_epf *epf_vf)
{
    u8 func_no, vfunc_no;
    int ret;

    mutex_lock(&epf_pf->lock);

    // 1. 检查 VF 数量限制
    vfunc_no = epf_vf->vfunc_no;
    func_no = epf_pf->func_no;

    if (vfunc_no > 0) {
        if (!epf_pf->epc || !epf_pf->epc->max_vfs) {
            ret = -EINVAL;
            goto ret;
        }

        if (vfunc_no > epf_pf->epc->max_vfs[func_no]) {
            ret = -EINVAL;
            goto ret;
        }
    }

    // 2. 设置 VF 的父物理函数
    epf_vf->epf_pf = epf_pf;

    // 3. 添加到父函数的 VF 列表
    list_add_tail(&epf_vf->pci_vepf_link, &epf_pf->pci_vepf);

    mutex_unlock(&epf_pf->lock);
    return epf_vf;

ret:
    mutex_unlock(&epf_pf->lock);
    return ERR_PTR(ret);
}
```

## EPC 内存管理 (pci-epc-mem.c)

### 初始化 EPC 内存窗口

#### pci_epc_multi_mem_init() - 初始化多窗口 (pci-epc-mem.c:47)

```c
int pci_epc_multi_mem_init(struct pci_epc *epc,
                           struct pci_epc_mem_window *windows,
                           unsigned int num_windows)
{
    struct pci_epc_mem *mem = NULL;
    unsigned long *bitmap = NULL;
    unsigned int page_shift;
    size_t page_size;
    int bitmap_size;
    int pages;
    int ret;
    int i;

    epc->num_windows = 0;

    if (!windows || !num_windows)
        return -EINVAL;

    // 1. 分配窗口数组
    epc->windows = kcalloc(num_windows, sizeof(*epc->windows), GFP_KERNEL);
    if (!epc->windows)
        return -ENOMEM;

    // 2. 为每个窗口初始化
    for (i = 0; i < num_windows; i++) {
        page_size = windows[i].page_size;
        if (page_size < PAGE_SIZE)
            page_size = PAGE_SIZE;
        page_shift = ilog2(page_size);

        // 计算页数和位图大小
        pages = windows[i].size >> page_shift;
        bitmap_size = BITS_TO_LONGS(pages) * sizeof(long);

        // 分配 pci_epc_mem 结构
        mem = kzalloc(sizeof(*mem), GFP_KERNEL);
        if (!mem) {
            ret = -ENOMEM;
            goto err_mem;
        }

        // 分配位图
        bitmap = kzalloc(bitmap_size, GFP_KERNEL);
        if (!bitmap) {
            ret = -ENOMEM;
            kfree(mem);
            goto err_mem;
        }

        // 初始化内存窗口
        mem->window.phys_base = windows[i].phys_base;
        mem->window.size = windows[i].size;
        mem->window.page_size = page_size;
        mem->bitmap = bitmap;
        mem->pages = pages;
        mutex_init(&mem->lock);

        epc->windows[i] = mem;
    }

    epc->mem = epc->windows[0];  // 设置默认窗口
    epc->num_windows = num_windows;

    return 0;

err_mem:
    // 清理已分配的窗口
    for (; i >= 0; i--) {
        mem = epc->windows[i];
        kfree(mem->bitmap);
        kfree(mem);
    }
    kfree(epc->windows);

    return ret;
}
```

### 分配 EPC 地址空间

#### pci_epc_mem_alloc_addr() - 分配地址 (pci-epc-mem.c)

```c
void __iomem *pci_epc_mem_alloc_addr(struct pci_epc *epc,
                                    phys_addr_t *phys_addr,
                                    size_t size,
                                    size_t *offset)
{
    struct pci_epc_mem *mem = epc->mem;
    unsigned long pfn_start;
    size_t page_size;
    unsigned int bitmap_size;
    int pfn;
    int order;
    int ret;

    mutex_lock(&mem->lock);

    // 1. 计算需要的页数
    page_size = mem->window.page_size;
    order = pci_epc_mem_get_order(mem, size);
    pfn_start = -1;

    // 2. 在位图中查找连续的页
    bitmap_size = mem->pages;
    ret = bitmap_find_free_region(mem->bitmap, bitmap_size,
                                  order, &pfn_start);

    // 3. 标记这些页为已使用
    for (pfn = pfn_start; pfn < pfn_start + (1 << order); pfn++)
        set_bit(pfn, mem->bitmap);

    mutex_unlock(&mem->lock);

    // 4. 计算物理地址和偏移
    *phys_addr = mem->window.phys_base + (pfn_start * page_size);
    *offset = *phys_addr - mem->window.phys_base;

    // 5. 返回虚拟地址（由平台驱动提供）
    return mem->window.virt_base + offset;
}
```

### 释放 EPC 地址空间

#### pci_epc_mem_free_addr() - 释放地址 (pci-epc-mem.c)

```c
void pci_epc_mem_free_addr(struct pci_epc *epc, phys_addr_t phys_addr)
{
    struct pci_epc_mem *mem = epc->mem;
    size_t offset;
    unsigned int pfn, pfn_start;
    int order;
    int size;

    mutex_lock(&mem->lock);

    // 1. 计算页帧号
    offset = phys_addr - mem->window.phys_base;
    pfn = offset >> ilog2(mem->window.page_size);
    pfn_start = pfn;

    // 2. 计算页数
    size = 1 << pci_epc_mem_get_order(mem, size);

    // 3. 清除位图中的这些页
    for (pfn = pfn_start; pfn < pfn_start + size; pfn++)
        clear_bit(pfn, mem->bitmap);

    mutex_unlock(&mem->lock);
}
```

## Endpoint Function 驱动示例

### Test 功能驱动 (pci-epf-test.c)

**用途**：提供测试功能，用于验证 PCIe Endpoint 框架的正确性

**功能**：
- BAR 读写测试
- 内存到内存数据传输
- DMA 传输
- Legacy INTx、MSI、MSI-X 中断
- Doorbell 机制

**关键结构**：

```c
struct pci_epf_test {
    void            *reg[PCI_STD_NUM_BARS];  /* BAR 寄存器 */
    struct pci_epf   *epf;                   /* EPF 设备 */
    enum pci_barno  test_reg_bar;            /* 测试寄存器 BAR */
    size_t           msix_table_offset;        /* MSI-X 表偏移 */
    struct delayed_work cmd_handler;            /* 命令处理 */
    struct dma_chan   *dma_chan_tx;           /* DMA TX 通道 */
    struct dma_chan   *dma_chan_rx;           /* DMA RX 通道 */
    struct dma_chan   *transfer_chan;         /* DMA 传输通道 */
    dma_cookie_t    transfer_cookie;        /* DMA Cookie */
    enum dma_status  transfer_status;        /* DMA 传输状态 */
    struct completion transfer_complete;      /* 完成通知 */
    bool            dma_supported;          /* DMA 支持标志 */
    bool            dma_private;            /* DMA 私有标志 */
    const struct pci_epc_features *epc_features;  /* EPC 能力 */
    struct pci_epf_bar db_bar;               /* Doorbell BAR */
};

/* 测试寄存器布局 */
struct pci_epf_test_reg {
    __le32 magic;
    __le32 command;       /* 命令位图 */
    __le32 status;        /* 状态位图 */
    __le64 src_addr;      /* 源地址 */
    __le64 dst_addr;      /* 目的地址 */
    __le32 size;
    __le32 checksum;
    __le32 irq_type;      /* 中断类型 */
    __le32 irq_number;    /* 中断号 */
    __le32 flags;
    __le32 caps;          /* 能力 */
    __le32 doorbell_bar;  /* Doorbell BAR */
    __le32 doorbell_offset;
    __le32 doorbell_data;
} __packed;
```

**命令定义**：
- COMMAND_RAISE_INTX_IRQ - 触发 Legacy INTx 中断
- COMMAND_RAISE_MSI_IRQ - 触发 MSI 中断
- COMMAND_RAISE_MSIX_IRQ - 触发 MSI-X 中断
- COMMAND_READ - 执行读操作
- COMMAND_WRITE - 执行写操作
- COMMAND_COPY - 执行内存复制
- COMMAND_ENABLE_DOORBELL - 使能 Doorbell

**状态定义**：
- STATUS_READ_SUCCESS/FAIL - 读操作结果
- STATUS_WRITE_SUCCESS/FAIL - 写操作结果
- STATUS_IRQ_RAISED - 中断已触发
- STATUS_DOORBELL_SUCCESS - Doorbell 操作成功

### NTB 功能驱动 (pci-epf-ntb.c) - EP 控制器间桥接

**用途**：实现 Non-Transparent Bridge (NTB)，在同一个 SoC 上连接两个不同的 PCIe Endpoint Controller

**应用场景**：
- SoC 有多个 PCIe Endpoint Controller 实例
- HOST1 通过 EP CONTROLLER1 与 SoC() 通信
- HOST2 通过 EP CONTROLLER2 与 SoC() 通信
- SoC() 作为桥接，实现 EP CONTROLLER1 ↔ EP CONTROLLER2 之间的通信

**拓扑结构**：
```
    +-------------+                                   +-------------+
    |             |                                   |             |
    |    HOST1    |                                   |    HOST2    |
    |             |                                   |             |
    +------^------+                                   +------^------+
           |                                                 |
           |                                                 |
    +---------|-------------------------------------------------|---------+
    |  +------v------+                                   +------v------+  |
    |  |             |                                   |             |  |
    |  |     EP      |                                   |     EP      |  |
    |  | CONTROLLER1 |                                   | CONTROLLER2 |  |
    |  |             <----------------------------------->             |  |
    |  |             |                                   |             |  |
    |  |             |                                   |             |  |
    |  |             | SoC() With Multiple EP Instances   |             |  |
    |  +-------------+                                   +-------------+  |
    +---------------------------------------------------------------------+
```

**关键结构**：

```c
/* BAR 类型定义 */
enum epf_ntb_bar {
    AR_CONFIG,      /* 配置 BAR */
    AR_PEER_SPAD,    /* 对端 Scratchpad 地址空间 */
    AR_DB_MW1,       /* Doorbell + MW1 BAR */
    AR_MW2,          /* MW2 BAR */
    AR_MW3,          /* MW3 BAR */
    AR_MW4,          /* MW4 BAR */
};

/* NTB EPF 结构 */
struct epf_ntb {
    u32 num_mws;                    /* Memory Window 数量 */
    u32 db_count;                   /* Doorbell 数量 */
    u32 spad_count;                 /* Scratchpad 地址数 */
    struct pci_epf *epf;            /* EPF 设备 */
    u64 mws_size[MAX_MW];           /* 每个 MW 的大小 */
    struct config_group group;         /* ConfigFS 组 */

    /* 指向两个 EPC */
    struct epf_ntb_epc *epc[2];      /* PRIMARY 和 SECONDARY EPC */
};

/* 每个 EPC 的信息 */
struct epf_ntb_epc {
    u8 func_no;                     /* 函数号 */
    u8 vfunc_no;                    /* 虚拟函数号 */
    bool linkup;                     /* 链路状态 */
    bool is_msix;                    /* 使用 MSI-X */
    int msix_bar;                    /* MSI-X BAR */
    u32 spad_size;                   /* Scratchpad 大小 */
    struct pci_epc *epc;             /* EPC 设备 */
    struct epf_ntb *epf_ntb;          /*(父 NTB EPF */
    void __iomem *mw_addr[6];         /* Memory Window 地址 */
    size_t msix_table_offset;         /* MSI-X 表偏移 */
    struct epf_ntb_ctrl *reg;        /* 控制寄存器 */
    struct pci_epf_bar *epf_bar;      /* EPF BAR */
    enum pci_barno epf_ntb_bar[6];   /* BAR 编号 */
    struct delayed_work cmd_handler;   /* 命令处理 */
    enum pci_epc_interface_type type;    /* PRIMARY/SECONDARY */
    const struct pci_epc_features *epc_features;  /* EPC 能力 */
};
```

**配置头**：

```c
static struct pci_epf_header epf_ntb_header = {
    .vendorid    = PCI_ANY_ID,
    .deviceid    = PCI_ANY_ID,
    .baseclass_code = PCI_BASE_CLASS_MEMORY,  /* Memory Device */
    .interrupt_pin  = PCI_INTERRUPT_INTA,
};
```

**关键功能**：

1. **epf_ntb_link_up()** - 触发链路状态中断
```c
static int epf_ntb_link_up(struct epf_ntb *ntb, bool link_up)
{
    enum pci_epc_interface_type type;
    struct epf_ntb_epc *ntb_epc;
    struct epf_ntb_ctrl *ctrl;
    unsigned int irq_type;
    struct pci_epc *epc;
    u8 func_no, vfunc_no;
    bool is_msix;
    int ret;

    // 向两个 EPC 都触发中断
    for (type = PRIMARY_INTERFACE; type <= SECONDARY_INTERFACE; type++) {
        ntb_epc = ntb->epc[type];
        epc = ntb_epc->epc;
        func_no = ntb_epc->func_no;
        vfunc_no = ntb_epc->vfunc_no;
        is_msix = ntb_epc->is_msix;
        ctrl = ntb_epc->reg;

        if (link_up)
            ctrl->link_status |= LINK_STATUS_UP;
        else
            ctrl->link_status &= ~LINK_STATUS_UP;

        irq_type = is_msix ? PCI_IRQ_MSIX : PCI_IRQ_MSI;
        ret = pci_epc_raise_irq(epc, func_no, vfunc_no, irq_type, 1);
        if (ret) {
            dev_err(&epc->dev,
                    "%s intf: Failed to raise Link Up IRQ\n",
                    pci_epc_interface_string(type));
            return ret;
        }
    }

    return 0;
}
```

2. **epf_ntb_configure_mw()** - 配置出站地址空间
```c
static int epf_ntb_configure_mw(struct epf_ntb *ntb, u32 mw)
{
    // 映射本地内存窗口到对端的 PCIe 地址空间
    ret = pci_epc_map_addr(ntb->epf->epc, func_no, vfunc_no, phys_addr, addr, size);
}
```

### Virtual NTB 功能驱动 (pci-epf-vntb.c) - RC↔EP 桥接

**用途**：在 PCIe Root Complex 和 PCIe Endpoint 之间实现的 Virtual NTB

**应用场景**：
- SoC() 作为 Virtual NTB 桥接器
- PCI RC 通过 Virtual NTB 与 PCI EP() 通信
- 实现透明/非透明桥接功能

**拓扑结构**：
```
    +------------+         +---------------------------------------+
    |            |         |                                       |
    +------------+         |                        +--------------+
    | NTB        |         |                        | NTB          |
    | NetDev     |         |                        | NetDev       |
    +------------+         |                        +--------------+
    | NTB        |         |                        | NTB          |
    | Transfer   |         |                        | Transfer     |
    +------------+         |                        +--------------+
    |            |         |                        |              |
    |  PCI NTB   |         |                        |              |
    |    EPF     |         |                        | PCI Virtual  |
    |   Driver   |         |                        | NTB Driver   |
    |            |         | +---------------+        |              |
    |            |         | PCI EP NTB    |<------>|              |
    |            |         |   FN Driver    |        |              |
    +------------+         +---------------+        +--------------+
    |            |         |               |        |              |
    |  PCI Bus   | <-----> |  PCI EP Bus   |        | Virtual PCI |
    |            |  PCI    |               |        |     Bus      |
    +------------+         +---------------+--------+--------------+
    PCIe Root Port                        PCI EP
```

**与 pci-epf-ntb.c 的区别**：

| 特性 | pci-epf-ntb.c | pci-epf-vntb.c |
|------|---------------|---------------|
| 连接对象 | EP CONTROLLER1 ↔ EP CONTROLLER2 | PCI RC ↔ PCI EP() |
| 桥接位置 | 同一 SoC() 内部（多 EP() 实例） | SoC() 在两个外部 PCIe 总线之间 |
| 用途 | EP() 间数据转发 | RC↔EP 透明桥接 |
| EPC 接口 | PRIMARY 和 SECONDARY | 仅 PRIMARY (使用 Virtual NTB) |

### MHI 功能驱动 (pci-epf-mhi.c)

**用途**：实现 MHI (Modem Host Interface)，用于与调制解调器通信

**应用场景**：
- SoC() 作为调制解调器的 PCIe Endpoint
- 与主机系统进行高速数据通信
- 支持多通道协议（SAHARA、DIAG、QSS 等）

**关键结构**：

```c
#define MHI_EP_CHANNEL_CONFIG_UL(ch_num, ch_name)    \
    {                                 \
        .num = ch_num,                    \
        .name = ch_name,                \
        .dir = DMA_TO_DEVICE              \
    }

#define MHI_EP_CHANNEL_CONFIG_DL(ch_num, ch_name)    \
    MHI_EP_CHANNEL_CONFIG(ch_num, ch_name, DMA_FROM_DEVICE)

/* MHI V1 通道配置 */
static const struct mhi_ep_channel_config mhi_v1_channels[] = {
    MHI_EP_CHANNEL_CONFIG_UL(0, "LOOPBACK"),
    MHI_EP_CHANNEL_CONFIG_DL(1, "LOOPBACK"),
    MHI_EP_CHANNEL_CONFIG_UL(2, "SAHARA"),
    MHI_EP_CHANNEL_CONFIG_DL(3, "SAHARA"),
    MHI_EP_CHANNEL_CONFIG_UL(4, "DIAG"),
    MHI_EP_CHANNEL_CONFIG_DL(5, "DIAG"),
    /* ... 更多通道 ... */
};
```

## 配置空间管理

### EPC 写入配置头

平台驱动实现的 `write_header` 操作：

```c
static int designware_pcie_ep_write_header(struct pci_epc *epc, u8 func_no,
                                        u8 vfunc_no,
                                        struct pci_epf_header *hdr)
{
    struct dw_pcie_ep *ep = to_dw_pcie_ep_from_epc(epc);
    u32 reg;

    // 1. 写入 Vendor ID
    reg = hdr->vendorid;
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                         PCIE_CONFIG VendorId, reg);

    // 2. 写入 Device ID
    reg = hdr->deviceid;
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                         PCIE_CONFIG DeviceId, reg);

    // 3. 写入 Class Code
    reg = (hdr->baseclass_code << 24) |
           (hdr->subclass_code << 16) |
           (hdr->progif_code << 8) |
           hdr->revid;
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                         PCIE_CONFIG ClassCode, reg);

    // ... 写入其他寄存器 ...

    return 0;
}
```

**DDBI (双字节配置空间接口)**：
- 通过 Direct Byte Configuration Access 写入配置空间
- 一次写入 32 位，需要处理字节序

### 配置 BAR

平台驱动实现的 `set_bar` 操作：

```c
static int designware_pcie_ep_set_bar(struct pci_epc *epc, u8 func_no,
                                   u8 vfunc_no,
                                   struct pci_epf_bar *epf_bar)
{
    struct dw_pcie_ep *ep = to_dw_pcie_ep_from_epc(epc);
    u32 reg;
    size_t size;
    int bar = epf_bar->barno;

    // 1. 写入 BAR 地址（低 32 位）
    reg = lower_32_bits(epf_bar->phys_addr);
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                         PCIE_CONFIG_BAR0 + bar * 4, reg);

    // 2. 如果是 64 位 BAR，写入高 32 位
    if (epf_bar->flags & PCI_BASE_ADDRESS_MEM_TYPE_64) {
        reg = upper_32_bits(epf_bar->phys_addr);
        dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_BAR0 + (bar + 1) * 4, reg);
    }

    // 3. 配置 BAR 大小（通过 BAR Mask）
    size = pci_epc_get_bar_size(epf_bar->size);
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                         PCIE_CONFIG_BAR0 + bar * 4, size);

    return 0;
}
```

## 地址映射

### 映射 CPU 地址到 PCIe 地址

EPC 驱动的 `map_addr` 操作：

```c
static int designware_pcie_ep_map_addr(struct pci_epc *epc, u8 func_no,
                                     u8 vfunc_no,
                                     phys_addr_t addr,
                                     u64 pci_addr, size_t size)
{
    struct dw_pcie_ep *ep = to_dw_pcie_ep_from_epc(epc);
    struct dw_pcie_ep_func *ep_func;
    u32 atu_type;

    ep_func = &ep->func[func_no];

    // 1. 确定 ATU (Address Translation Unit) 类型
    if (addr < ep->mem.phys_base) {
        // 内部内存到 PCIe 地址
        atu_type = PCIE_ATU_TYPE_INBOUND;
    } else {
        // PCIe 地址到内部内存
        atu_type = PCIE_ATU_TYPE_OUTBOUND;
    }

    // 2. 配置 ATU 索引
    dw_pcie_ep_configure_atu(ep, ep_func, atu_type,
                             func_no, vfunc_no,
                             addr, pci_addr, size);

    return 0;
}
```

**ATU (Address Translation Unit)**：
- PCIe Endpoint() 中的硬件功能
- 将 CPU 物理地址映射到 PCIe 地址空间
- 或将 PCIe 地址映射到 CPU 物理地址

**示例流程**：
```
1. EPF() 驱动分配本地缓冲区 (malloc/dma_alloc_coherent)
2. 调用 pci_epc_mem_alloc_addr() 获取 EPC() 地址
3. 调用 pci_epc_map_addr() 建立映射
4. RC 可以通过 PCIe() 地址访问 EPF() 的内存
```

## 中断处理

### 配置 MSI

EPC 驱动的 `set_msi` 操作：

```c
static int designware_pcie_ep_set_msi(struct pci_epc *epc, u8 func_no,
                                    u8 vfunc_no,
                                    u8 nr_irqs)
{
    struct dw_pcie_ep *ep = to_dw_pcie_ep_from_epc(epc);
    struct dw_pcie_ep_func *ep_func = &ep->func[func_no];
    u32 val;

    // 1. 启用 MSI 能力
    val = dw_pcie_ep_readl_dbi(ep, func_no, vfunc_no,
                              PCIE_CONFIG_MSI_CAP);
    val |= MSI_CAP_MSI_ENABLE;
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSI_CAP, val);

    // 2. 设置可分配的中断数量
    val = (nr_irqs & MSI_CAP_MME_MASK) << MSI_CAP_MME_SHIFT;
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSI_CAP, val);

    return 0;
}
```

### 配置 MSI-X

EPC 驱动的 `set_msix` 操作：

```c
static int designware_pcie_ep_set_msix(struct pci_epc *epc, u8 func_no,
                                     u8 vfunc_no,
                                     u16 nr_irqs,
                                     enum pci_barno barno,
                                     u32 offset)
{
    struct dw_pcie_ep *ep = to_dw_pcie_ep_from_epc(epc);
    struct dw_pcie_ep_func *ep_func = &ep->func[func_no];
    u32 val;

    // 1. 启用 MSI-X 能力
    val = dw_pcie_ep_readl_dbi(ep, func_no, vfunc_no,
                              PCIE_CONFIG_MSIX_CAP);
    val |= MSI_CAP_MSIX_ENABLE;
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSIX_CAP, val);

    // 2. 配置 MSI-X 表大小
    val = ((nr_irqs - 1) & MSIX_TABLE_SIZE_MASK);
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSIX_CAP, val);

    // 3. 设置 MSI-X 表位置 (BIR + Offset)
    val = barno | offset;
    dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSIX_CAP, val);

    return 0;
}
```

### 触发中断

EPC 驱动的 `raise_irq` 操作：

```c
static int designware_pcie_ep_raise_irq(struct pci_epc *epc, u8 func_no,
                                     u8 vfunc_no,
                                     unsigned int type,
                                     u16 interrupt_num)
{
    struct dw_pcie_ep *ep = to_dw_pcie_ep_from_epc(epc);
    struct dw_pcie_ep_func *ep_func = &ep->func[func_no];
    u32 reg;

    switch (type) {
    case PCI_IRQ_LEGACY:
        // 触发 Legacy INTx
        reg = BIT(interrupt_num);
        dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_CMD, reg);
        break;

    case PCI_IRQ_MSI:
        // 触发 MSI
        reg = BIT(interrupt_num);
        dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSI_DATA, reg);
        // 触发中断请求
        dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSI_INT, BIT(0));
        break;

    case PCI_IRQ_MSIX:
        // 触发 MSI-X
        reg = interrupt_num;
        dw_pcie_ep_writel_dbi(ep, func_no, vfunc_no,
                             PCIE_CONFIG_MSIX_TABLE, reg);
        break;
    }

    return 0;
}
```

## Doorbell 机制

**用途**：Doorbell 是 PCIe Endpoint() 向 Root Complex 通知有数据的机制

**工作原理**：
1. Endpoint() 准备数据到共享内存区域
2. Endpoint() 通过 Doorbell() 写入特定值到 Doorbell() 寄存器
3. Doorbell() 写操作触发 MSI-X 中断到 RC
4. RC 中断处理程序读取 Doorbell() 值
5. RC 根据 Doorbell() 值判断哪个 EPF() 有数据

**相关文件**：
- `pci-ep-msi.c` - MSI Doorbell() 支持
- `pci-epf-test.c` - 测试功能中的 Doorbell() 实现
- `pci-epf-ntb.c` - NTB 功能中的 Doorbell()

**流程示例**：
```
EPF() 驱动:
1. 准备数据到共享缓冲区
2. 写入 Doorbell() 寄存器: value = 0x1234
3. pci_epc_raise_irq(epc, func_no, 0, PCI_IRQ_MSIX, doorbell_irq_num)

RC 端:
1. 收到 MSI-X 中断
2. 读取 Doorbell() 寄存器: value = 0x1234
3. 根据 value 处理 EPF() 数据
```

## 典型 EPF 驱动开发流程

### EPF 驱动框架

```c
/* 1. 定义 EPF 设备 ID */
static const struct pci_epf_device_id my_epf_id_table[] = {
    { .name = "my_epf", .func = "my_epf_bind" },
    { },
};

/* 2. 定义配置头 */
static struct pci_epf_header my_epf_header = {
    .vendorid      = 0x1234,           /* 自定义 Vendor ID */
    .deviceid      = 0x5678,           /* 自定义 Device ID */
    .baseclass_code = PCI_CLASS_OTHERS,
    .interrupt_pin  = PCI_INTERRUPT_INTA,
};

/* 3. 定义 bind 回调 */
static int my_epf_bind(struct pci_epf *epf)
{
    struct pci_epc *epc = epf->epc;
    int bar;

    // 获取 EPC 能力
    epf->epc_features = pci_epc_get_features(epc, epf->func_no, 0);

    // 查找可用的 BAR
    bar = pci_epc_get_first_free_bar(epf->epc_features);

    // 分配 BAR 空间
    epf->bar[bar].phys_addr = pci_epc_mem_alloc_addr(epc,
                                            &epf->bar[bar].phys_addr,
                                            BAR_SIZE,
                                            &epf->bar[bar].size);
    epf->bar[bar].barno = bar;

    // 配置 BAR
    pci_epc_set_bar(epf, &epf->bar[bar]);

    // 配置 MSI-X
    epf->msix_interrupts = 32;
    pci_epc_set_msix(epf, epf->msix_interrupts, BAR_0, 0);

    // 写入配置头
    pci_epc_write_header(epf, &my_epf_header);

    return 0;
}

/* 4. 定义 unbind 回调 */
static void my_epf_unbind(struct pci_epf *epf)
{
    int bar;

    // 清除 BAR
    bar = epf->bar[0].barno;
    pci_epc_clear_bar(epf, &epf->bar[bar]);

    // 释放 BAR 空间
    pci_epc_mem_free_addr(epc, epf->bar[bar].phys_addr);
}

/* 5. 定义 add_cfs 回调 (可选，添加 ConfigFS 属性) */
static struct config_group *my_epf_add_cfs(struct pci_epf *epf,
                                           struct config_group *group)
{
    // 添加功能特定的 ConfigFS 属性
    return NULL;
}

/* 6. 定义 EPF 操作 */
static const struct pci_epf_ops my_epf_ops = {
    .bind   = my_epf_bind,
    .unbind = my_epf_unbind,
    .add_cfs = my_epf_add_cfs,
};

/* 7. 定义 probe 回调 */
static int my_epf_probe(struct pci_epf *epf,
                       const struct pci_epf_device_id *id)
{
    // 分配私有数据
    struct my_epf_data *data = kzalloc(sizeof(*data), GFP_KERNEL);
    epf->driver_private = data;

    return 0;
}

/* 8. 定义 remove 回调 */
static void my_epf_remove(struct pci_epf *epf)
{
    // 释放私有数据
    kfree(epf->driver_private);
}

/* 9. 定义 EPF 驱动 */
static struct pci_epf_driver my_epf_driver = {
    .driver.name = KBUILD_MODNAME,
    .probe   = my_epf_probe,
    .remove  = my_epf_remove,
    .ops     = &my_epf_ops,
    .owner    = THIS_MODULE,
    .id_table = my_epf_id_table,
};

/* 10. 注册驱动 */
module_pci_epf_driver(my_epf_driver);
```

## ConfigFS 接口

### ConfigFS 目录结构

```
/config/pci-epc/
├── <epc_name>/            # 每个 EPC 设备的目录
│   ├── <epf_name>/       # 每个 EPF 设备的目录
│   │     ├── start        # 启动链路
│   │   ├── stop         # 停止链路
│   │   ├── vendor       # Vendor ID
│   │   ├── device       # Device ID
│   │   ├── revid        # Revision ID
│   │   ├── ...          # 其他配置属性
│   │   └── functions/   # 功能特定目录
│   │       ├── ...       # 功能特定属性
```

### 典型使用流程

```bash
# 1. 创建 EPC
mkdir /config/pci-epc/pcie-ep

# 2. 添加 EPF
mkdir /config/pci-epc/pcie-ep/ntb

# 3. 配置 EPF
echo 0x1234 > /config/pci-epc/pcie-ep/ntb/vendor
echo 0x5678 > /config/pci-epc/pcie-ep/ntb/device

# 4. 启动链路
echo 1 > /config/pci-epc/pcie-ep/start
```

## 总结

### PCIe Endpoint() 框架关键特性

1. **双角色支持**
   - SoC() 可以作为 PCIe Endpoint() 或 Root Complex
   - 动态切换角色（需要平台驱动支持）

2. **函数和虚拟函数**
   - 支持多个物理函数 (0, 1, 2, ...)
   - 支持 SR-IOV 虚拟函数 (SR-IOV 1.0)
   - 每个函数独立配置

3. **内存地址空间管理**
   - 多窗口支持
   - 位图管理的地址分配
   - 自动页对齐

4. **地址映射**
   - ATU (Address Translation Unit) 支持
   - CPU 物理地址 ↔ PCIe 地址映射
   - 支持入站和出站映射

5. **中断支持**
   - Legacy INTx
   - MSI (Message Signaled Interrupts)
   - MSI-X (Extended MSI)
   - Doorbell() 机制

6. **ConfigFS 接口**
   - 运行时配置
   - 无需修改内核代码
   - 易于调试和测试

### NTB (Non-Transparent Bridge) 两种模式

| 模式 | 驱动 | 拓扑 | 用途 |
|------|------|------|------|
| EP() 间桥接 | pci-epf-ntb.c | EP CONTROLLER1 ↔ EP CONTROLLER2 | 同一 SoC() 上多 EP() 实例间通信 |
| Virtual NTB | pci-epf-vntb.c | PCI RC ↔ PCI EP() | RC() 和 EP() 之间透明桥接 |

### 典型应用场景

1. **网络加速器**
   - SoC() 作为网络 PCIe() 卡
   - 提供硬件加速的网络功能

2. **存储控制器**
   - SoC() 作为存储 PCIe() 卡
   - NVMe/SCSI 协议实现

3. **多主机桥接**
   - NTB 功能连接多个 PCIe 域
   - 实现透明/非透明桥接

4. **调制解调器**
   - MHI 功能与调制解调器通信
   - 高速数据传输

### 相关文件清单

| 文件 | 功能 |
|------|------|
| pci-epc-core.c | EPC 核心库 |
| pci-epf-core.c | EPF 核心库 |
| pci-epc-mem.c | EPC 内存管理 |
| pci-ep-cfs.c | ConfigFS 接口 |
| pci-ep-msi.c | MSI(Doorbell) |
| pci-epf-test.c | 测试功能驱动 |
| pci-epf-ntb.c | EP() 间 NTB 桥接 |
| pci-epf-vntb.c | RC()↔EP Virtual NTB |
| pci-epf-mhi.c | MHI 功能驱动 |

### 平台驱动示例

| 平平台 | 驱动文件 | 说明 |
|------|---------|------|
| DesignWare | dw-pcie-ep.c | Synopsys DesignWare PCIe Endpoint() |
| Cadence | cdns-pcie-ep.c | Cadence PCIe Endpoint() |
| Rockchip | rockchip-pcie-ep.c | Rockchip SoC PCIe Endpoint() |
| TI | keystone-pcie-ep.c | Texas Instruments Keystone PCIe Endpoint() |
