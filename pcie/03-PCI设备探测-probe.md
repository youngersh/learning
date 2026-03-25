# PCI设备探测分析 (probe.c)

## 文件概述
`probe.c` 实现PCI设备的检测、发现和初始化，是PCI枚举的核心文件。

## 核心数据结构

### pci_root_buses
```c
LIST_HEAD(pci_root_buses);
```
- 全局链表，存储所有PCI根总线

### pci_domain_busn_res
```c
struct pci_domain_busn_res {
    struct list_head list;
    struct resource res;
    int domain_nr;
};
```
- 用于管理PCI域的总线号资源

## 主要功能函数

### 1. 总线状态查询

#### no_pci_devices
检查系统是否有PCI设备：

```c
int no_pci_devices(void)
```

**实现逻辑：**
- 遍历 `pci_bus_type` 设备
- 如果没有找到设备返回 `true`

### 2. 总线类管理

#### pcibus_dev_release
PCI总线设备的释放函数：

```c
static void release_pcibus_dev(struct device *dev)
```

**清理内容：**
- 释放总线资源: `pci_bus_remove_resources()`
- 释放设备树节点: `pci_release_bus_of_node()`
- 释放 `pci_bus` 结构

### 3. 总线创建

#### pci_alloc_bus
分配新的PCI总线：

```c
struct pci_bus *pci_alloc_bus(struct pci_bus *parent, int devfn,
                             struct pci_ops *ops, void *sysdata)
```

**初始化步骤：**
1. 分配 `pci_bus` 结构
2. 设置父总线和操作指针
3. 初始化设备链表
4. 设置总线号（如果是根总线）
5. 创建总线kobject

#### pci_add_bus
添加PCI总线到系统：

```c
struct pci_bus *pci_add_bus(struct pci_bus *parent, int devfn,
                          struct pci_ops *ops, void *sysdata, int bus)
```

**处理流程：**
1. 查找或分配host bridge
2. 分配总线结构
3. 设置总线号
4. 添加到全局总线列表
5. 初始化sysfs

### 4. 设备创建

#### pci_alloc_dev
分配新的PCI设备：

```c
struct pci_dev *pci_alloc_dev(struct pci_bus *bus, int devfn)
```

**初始化内容：**
- 设置总线和设备/功能号
- 初始化资源列表
- 初始化DMA掩码
- 设置默认状态

#### pci_scan_single_devicequisition
扫描单个设备：*

```c
struct pci_dev *pci_scan_single_device(struct pci_bus *bus, int devfn,
                                    int pass)
```

**扫描流程：**
1. 读取厂商ID和设备ID
2. 检查是否有效设备
3. 应用early fixups
4. 分配设备结构
5. 设置设备信息（头类型、类别等）
6. 保存配置空间
7. 应用early fixups
8. 设置总线主控和内存使能
9. 解析设备能力

### 5. 设备设置

#### pci_setup_device
初始化PCI设备：

```c
int pci_setup_device(struct pci_dev *dev)
```

**设置流程：**
1. 解析BAR（基址寄存器）
- `pci_read_base()` - 读取每个BAR
- `__pci_read_base()` - 设置资源类型和大小
2. 设置ROM资源
- `pci_enable_rom_bar()` - 处理ROM BAR
3. 设置缓存行大小
4. 检测并设置中断引脚
5. 应用quirks
6. 初始化PCIe能力
7. 设置MSI/MSI-X

### 6.1 总线扫描

#### pci_scan_slot
扫描PCI总线上的设备：*

```c
int pci_scan_slot(struct pci_bus *bus)
```

**扫描策略：**
- 标准PCI扫描：遍历devfn 0-255
- ARI（Alternative Routing ID）支持：扫描最多256个函数
- 跳过空slot以加速

#### pci_scan_bridge
扫描PCI-PCI桥接：*

```c
int pci_scan_bridge(struct pci_bus *bus, struct pci_dev *dev, int max)
```

**桥接处理流程：**
1. 获取总线号范围
2. 设置子总线号
3. 检查总线号冲突
4. 创建子总线
5. 扫描子总线
6. 设置桥接资源窗口

### 7. 资源分配

#### pci_bus_assign_resources
分配总线资源：*

```c
void pci_bus_assign_resources(const struct pci_bus *bus)
```

**分配阶段：**
1. 第1遍：记录所有资源大小
- `pci_bus_size_bridges()` - 桥接资源
- `size_resources()` - 设备资源
2. 第2遍：分配资源
- `pci_bus_assign_resources()` - 递归分配

## 设备能力检测

### pci_enable_msi
使能MSI（Message Signaled Interrupts）：

```c
void pci_enable_msi(struct pci_dev *dev)
```

**实现逻辑：**
- 检查设备是否支持MSI
- 设置MSI使能位
- 分配MSI中断

### pci_init_capabilities
初始化设备能力：*

```c
static void pci_init_capabilities(struct pci_dev *dev)
```

**能力检测顺序：**
1. PM (Power Management)
2. MSI/MSI-X
3. PCIe扩展能力（AER, VC, ACS, MFVC等）
4. PTM (Precision Time Measurement)
5. PASID (Process Address Space ID)
6. PRI (Page Request Interface)
7. ATS (Address Translation Service)
8. SR-IOV (Single Root I/O Virtualization)

## Fixup机制

### pci_fixup_device
应用设备特定修复：*

```c
void pci_fixup_device(enum pci_fixup_pass pass, struct pci_dev *dev)
```

**Fixup阶段：**
1. `PCI_FIXUP_EARLY` - 设备分配前
2. `PCI_FIXUP_HEADER` - Header读取后
3. `PCI_FIXUP_FINAL` - 最终设置前
4. `PCIFixUP_SUSPEND` - 挂起前
5. `PCI_FIXUP_RESUME` - 恢复时

## 关键要点

1. **总线扫描**：先扫描父总线，再扫描子总线
2. **资源分配**：两遍分配，先记录后分配
3. **设备发现**：通过读取配置空间头识别设备
4. **AR支持**：PCIe ARI允许每个端口最多256个函数
5. **能力检测**：按顺序检测各种PCIe扩展能力
6. **Fixup系统**：多个阶段的设备特定修复
