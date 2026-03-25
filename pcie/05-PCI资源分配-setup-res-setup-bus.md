# PCI资源分配分析 (setup-res.c & setup-bus.c)

## 文件概述
- `setup-res.c`: PCI设备资源（BAR）的管理
- `setup-bus.c`: PCI总线资源的分配和调整

## 核心数据结构

### pci_dev_resource (setup-bus.c)
```c
struct pci_dev_resource {
    struct list_head    list;        // 链表节点
    struct resource    *res;        // 资源指针
    struct pci_dev    *dev;        // 所属设备
    resource_size_t   start;        // 资源起始地址
    resource_size_t   end;          // 资源结束地址
    resource_size_t   add_size;     // 额外添加的大小
    resource_size_t   min_align;    // 最小对齐要求
    unsigned long    flags;       // 资源标志
};
```
用于跟踪设备资源分配。

## setup-res.c 主要功能

### 1. BAR更新

#### pci_std_update_resource
更新标准PCI BAR：

```c
static void pci_std_update_resource(struct pci_dev *dev, int resno)
```

**更新逻辑：**
1. 检查VF BAR（SR-IOV虚拟函数，只读）
2. 跳过未实现的BAR和空资源
3. 跳过固定资源（`IORESOURCE_PCI_FIXED`）
4. 转换CPU地址为总线地址
5. 设置BAR寄存器值
6. 对于64位BAR，更新高32位寄存器

**64位BAR特殊处理：**
- 更新前临时禁用内存解码
- 防止部分更新的BAR导致地址冲突

#### pci_update_resource
设备资源更新入口：

```c
void pci_update_resource(struct pci_dev *dev, int resno)
```

**分发逻辑：**
- `resno <= PCI_ROM_RESOURCE`: 标准BAR
- `resno` 是SR-IOV BAR: 调用 `pci_iov_update_resource()`

### 2. 资源声明

#### pci_claim_resource
声明设备资源：

```c
int pci_claim_resource(struct pci_dev *dev, int resource)
```

**声明条件：**
1. 资源必须已分配（不是 `IORESOURCE_UNSET`）
2. RAM中的shadow copy不需要声明
3. 使用 `request_resource()` 声明到系统
4. 处理资源冲突

### 3. ROM资源管理

#### pci_enable_rom/disable_rom
ROM解码控制：

```c
int pci_enable_rom(struct pci_dev *pdev)
void pci_disable_rom(struct pci_dev *pdev)
```

**ROM BAR特殊处理：**
- 某些设备禁用时ROM BAR读为0
- 启用/禁用通过设置/清除ROM BAR的使能位
- 支持shadow copy检测

#### pci_map_rom
映射PCI ROM到内核空间：

```c
void __iomem *pci_map_rom(struct pci_dev *pdev, size_t length)
```

**映射过程：**
1. 检查ROM大小
2. 使用 `ioremap()` 映射
3. 使能ROM解码
4. 验证ROM签名（0xAA55, 0x5043 ("PCI ROM"））

#### pci_get_rom_size
确定实际ROM镜像大小：

```c
static size_t pci_get_rom_size(struct pci_dev *pdev, void __iomem *rom, size_t size)
```

**检测逻辑：**
- 读取PCI数据结构（PDS）
- 验证"PCIR"签名
- 返回实际镜像长度

## setup-bus.c 主要功能

### 1. 资源列表管理

#### add_to_list/remove_from_list
资源跟踪列表操作：

```c
static int add_to_list(struct list_head *head, struct pci_dev *dev,
                   struct resource *res, resource_size_t add_size,
                                     resource_size_t min_align)
static void remove_from_list(struct list_head *head, struct resource *res)
```

**跟踪内容：**
- 资源起始/结束地址
- 额外大小和对齐要求
- 资源标志

### 2. 桥接资源大小计算

#### pci_bus_size_bridges
计算桥接所需资源窗口：

```c
void __pci_bus_size_bridges(struct pci_bus *bus,
                          struct list_head *realloc_head)
```

**计算逻辑：**
1. 遍历总线上的所有桥接
2. 检查子总线是否需要调整
3. 收集IO、Memory和Prefetch Memory范围
4. 累加额外大小和对齐要求
5. 递归处理子总线

### 3. 设备资源大小计算

#### size_resources
计算设备资源需求：

```c
static void size_resources(struct pci_bus *bus, struct pci_dev *dev)
```

**处理步骤：**
1. 对于每个BAR：
   - 检查是否实现
   - 记录资源大小
   - 记录对齐要求
2. 对于ROM资源：
   - 记录ROM大小（如果使能）

### 4. 资源分配

#### pci_bus_assign_resources
分配总线资源：

```c
void __pci_bus_assign_resources(const struct pci_bus *bus,
                             struct list_head *realloc_head)
```

**三遍分配：**

**第1遍历 - 重新对齐：**
```c
__pci_bus_assign_resources(bus, realloc_head)
```
- 对齐子总线资源
- 调整设备资源以匹配父桥接

**第2遍历 - 分配未分配资源：**
```c
pci_assign_unassigned_resources(bus, realloc_head)
```
- 仍未分配的资源分配到总线窗口

**第3遍历 - 最终调整：**
- 标记失败资源
- 更新BAR寄存器

#### pci_assign_unassigned_resources
分配未分配的设备资源：

```c
void pci_assign_unassigned_resources(struct pci_bus *bus,
                                 struct list_head *realloc_head)
```

**分配策略：**
1. 收集总线上未分配的资源
2. 按地址类型分组（IO、Memory）
3. 对每种类型：
   - 按大小降序排列
   - 从最小资源开始分配
   - 满足对齐要求
   - 追加额外大小到BAR

### 5. 桥接窗口更新

#### pci_setup_bridge_io/windows
设置桥接资源窗口：

```c
static void pci_setup_bridge_io(struct pci_bus *bus, struct resource *res,
                           resource_size_t size, unsigned long type)
static void pci_setup_bridge_windows(struct pci_bus *bus,
                                struct resource *io, resource resource *mem,
                                resource size_t prefmem)
```

**窗口配置：**
1. 计算总线地址范围
2. 设置IO/内存基址和限制寄存器
3. 设置命令寄存器使能解码
4. 配置64位寻址（如果需要）

### 6. 可调整BAR支持

#### pci_rebar_get_current_sizes
获取Resizable BAR当前大小：

```c
void pci_rebar_get_current_sizes(struct pci_dev *dev, int *sizes)
```

#### pci_rebar_set_sizes
设置Resizable BAR大小：

```c
int pci_rebar_set_sizes(struct pci_dev *dev, const int *sizes)
```

**重新大小流程：**
1. 禁用所有相关BAR
2. 设置新大小
3. 重新使能BAR
4. 验证配置

## 关键概念

### 1. BAR类型
- **I/O BAR**: 端口空间资源
- **Memory BAR**: 内存映射资源
  - 32位内存
  - 64位内存
  - Prefetchable内存
- **ROM BAR**: 扩展ROM资源

### 2. 资源状态
- **未分配**: `IORESOURCE_UNSET`
- **已分配**: 有有效地址范围
- **已声明**: 已添加到系统资源树

### 3. 桥接资源窗口
- **IO窗口**: 桥接的IO地址空间
- **非预取内存窗口**: 标准内存空间
- **预取内存窗口**: Prefetchable内存空间

### 4. 三遍分配
1. **重新对齐**: 调整资源对齐
2. **分配**: 分配地址空间
3. **更新**: 写入硬件配置

### 5. 桥接类型
- **PCI-PCI桥接**: 标准桥接
- **CardBus桥接**: 旧式PC卡桥接
- **Host桥接**: 根PCI域

## 关键要点

1. **64位BAR**: 需要两个连续32位寄存器
2. **对齐要求**: 资源必须满足最小对齐
3. **桥接窗口**: 限制子总线地址范围
4. **三遍分配**: 确保所有资源正确分配
5. **可调整BAR**: PCIe支持运行时BAR大小调整
6. **ROM特殊处理**: 某些设备有buggy ROM BAR
