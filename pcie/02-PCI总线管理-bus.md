# PCI总线管理分析 (bus.c)

## 文件概述
`bus.c` 实现PCI总线的资源管理和设备添加功能，是PCI资源分配的核心。

## 核心数据结构

### pci_bus_resource
```c
struct pci_bus_resource {
    struct list_head    list;   // 链表节点
    struct resource    *res;    // 资源指针
};
```
用于管理总线的额外资源（超过PCI_BRIDGE_RESOURCE_NUM的数量）。

## 主要功能函数

### 1. 资源管理

#### pci_add_resource/pci_add_resource_offset
向host bridge资源列表添加资源：

```c
void pci_add_resource(struct list_head *resources, struct resource *res)
void pci_add_resource_offset(struct list_head *resources, struct resource *res,
                         resource_size_t offset)
```

**实现细节：**
- 创建 `resource_entry` 并设置offset
- 添加到资源链表尾部

#### pci_bus_add_resource/remove_resource
总线级别的资源管理：

```c
void pci_bus_add_resource(struct pci_bus *bus, struct resource *res)
void pci_bus_remove_resource(struct pci_bus *bus, struct resource *res)
void pci_bus_remove_resources(struct pci_bus *bus)
```

**资源存储：**
- 前 `PCI_BRIDGE_RESOURCE_NUM` 个资源存储在 `bus->resource[]` 数组
- 额外资源存储在 `bus->resources` 链表

#### pci_bus_resource_n
获取总线资源：

```c
struct resource *pci_bus_resource_n(const struct pci_bus *bus, int n)
```

**查询顺序：**
1. 先查 `bus->resource[0..PCI_BRIDGE_RESOURCE_NUM-1]`
2. 再查 `bus->resources` 链表

### 2. 资源分配

#### pci_bus_alloc_resource
从父总线分配资源：

```c
int pci_bus_alloc_resource(struct pci_bus *bus, struct resource *res,
                     resource_size_t size, resource_size_t align,
                     resource_size_t min, unsigned long type_mask,
                     resource_alignf alignf, void *alignData)
```

**分配策略：**
1. 64位内存资源：优先高地址区域，失败后尝试低地址区域
2. 32位资源：仅在32位区域分配
3. 使用 `allocate_resource()` 在总线窗口中查找合适空间

**区域限制：**
- `pci_32_bit`: 0 ~ 0xffffffffULL (32位区域)
- `pci_64_bit`: 0 ~ 0xffffffffffffffffULL (64位区域)
- `pci_high`: 0x100000000 ~ 0xffffffffffffffffULL (高地址区域)

#### pci_bus_clip_resource
裁剪资源以适配父总线窗口：

```c
bool pci_bus_clip_resource(struct pci_dev *dev, int idx)
```

**处理逻辑：**
- 检查与父总线资源的重叠
- 裁剪到重叠范围内
- 标记为 `IORESOURCE_UNSET` 避免进一步修改

### 3. 设备添加和移除

#### pci_bus_add_device
添加单个设备到总线：

```c
void pci_bus_add_device(struct pci_dev *dev)
```

**初始化流程：**
1. 调用架构特定 `pcibios_bus_add_device()`
2. 应用最终fixup: `pci_fixup_device(pci_fixup_final, dev)`
3. 为PCIe桥接创建设备树节点
4. 创建sysfs和proc条目
5. 更新D3电源状态
6. 保存配置空间用于错误恢复
7. 处理pwrctrl设备链接
8. 执行设备探测: `device_initial_probe()`

#### pci_bus_add_devices
递归添加总线及其子总线的所有设备：

```c
void pci_bus_add_devices(const struct pci_bus *bus)
```

**遍历顺序：**
1. 先处理 `bus->devices` 链表中的所有设备
2. 递归处理每个设备的 `subordinate` 子总线

### 4. 总线遍历

#### pci_walk_bus
遍历总线及子总线的所有设备：

```c
void pci_walk_bus(struct pci_bus *top, int (*cb)(struct pci_dev *, void *),
                  void *userdata)
```

**遍历特性：**
- 使用读锁 `pci_bus_sem` 保护
- 前序遍历设备
- 对每个设备的子总线递归遍历

#### pci_walk_bus_reverse
反向遍历总线设备：

```c
void pci_walk_bus_reverse(struct pci_bus *top, int (*cb)(struct pci_dev *, void *),
                          void *userdata)
```

**反向遍历用途：**
- 按相反顺序处理设备（用于remove等操作）
- 先处理子总线，再处理父总线

### 5. 总线引用计数

#### pci_bus_get/pci_bus_put
总线生命周期管理：

```c
struct pci_bus *pci_bus_get(struct pci_bus *bus)
void pci_bus_put(struct pci_bus *bus)
```

**实现：**
- 调用 `get_device()/put_device()` 管理 `bus->dev`

## 架构回调

### pcibios_resource_survey_bus
```c
void __weak pcibios_resource_survey_bus(struct pci_bus *bus) { }
```
**weak函数**：允许架构提供自定义资源调查逻辑

### pcibios_bus_add_device
```c
void __weak pcibios_bus_add_device(struct pci_dev *pdev) { }
```
**weak函数**：允许架构执行设备添加时的特殊操作

## 辅助宏

### pci_bus_for_each_resource
```c
#define pci_bus_for_each_resource(bus, res)     \
    for (res = pci_bus_resource_n(bus, 0); \
         res != NULL;                    \
         res = pci_bus_resource_n(bus, (res->index) + 1))
```

**遍历所有资源：**
- 先遍历 `resource[]` 数组
- 再遍历 `resources` 链表

## 关键要点

1. **资源分层**：窗口资源在总线，设备资源在设备
2. **数组+链表**：固定资源用数组，动态资源用链表
3. **地址裁剪**：资源必须适配父总线窗口
4. **安全遍历**：使用 `pci_bus_sem` 读锁保护
5. **架构扩展**：通过weak函数允许架构特定行为
