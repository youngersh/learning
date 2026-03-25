# PCI设备移除分析 (remove.c)

## 文件概述
`remove.c` 实现PCI设备和总线的移除功能，处理设备热插拔和系统移除。

## 主要功能函数

### 1. 资源清理

#### pci_free_resources
释放设备的所有资源：

```c
static void pci_free_resources(struct pci_dev *dev)
```

**清理过程：**
1. 遍历所有设备资源
2. 如果资源有父节点（已声明），调用 `release_resource()`

### 2. 电源控制设备清理

#### pci_pwrctrl_unregister
注销电源控制设备：

```c
static void pci_pwrctrl_unregister(struct device *dev)
```

**清理流程：**
1. 获取设备的Device Tree节点
2. 查找对应的platform设备
3. 注销platform设备
4. 清除Device Tree节点的已创建标志

### 3. 设备停止

#### pci_stop_dev
停止PCI设备：

```c
static void pci_stop_dev(struct pci_dev *dev)
```

**停止操作：**
1. 禁用PME（Power Management Events）活动
2. 如果设备未添加过，直接返回
3. 释放设备驱动：`device_release_driver()`
4. 分离proc接口：`pci_proc_detach_device()`
5. 移除sysfs文件：`pci_remove_sysfs_dev_files()`
6. 移除Device Tree节点：`of_pci_remove_node()`

### 4. 设备销毁

#### pci_destroy_dev
销毁PCI设备：

```c
static void pci_destroy_dev(struct pci_dev *dev)
```

**销毁顺序：**
1. 设置设备已移除标志（防止重复销毁）
2. 销毁DOE sysfs：`pci_doe_sysfs_teardown()`
3. 移除NPEM：`pci_npem_remove()`
4. 退出TSM（Trusted Execution Environment）
5. 删除设备kobject：`device_del()`
6. 从总线列表移除设备（需 `pci_bus_sem` 写锁）
7. 销毁DOE：`pci_doe_destroy()`
8. 销毁IDE：`pci_ide_destroy()`
9. 退出ASPM链路状态
10. 更新桥接D3状态
11. 注销pwrctrl
12. 释放资源
13. 释放设备引用

### 5. 总线移除

#### pci_remove_bus
移除PCI总线：

```c
void pci_remove_bus(struct pci_bus *bus)
```

**移除流程：**
1. 分离proc总线接口
2. 从全局总线列表移除（需 `pci_bus_sem` 写锁）
3. 释放总线号资源：`pci_bus_release_busn_res()`
4. 移除legacy文件
5. 调用平台特定 `bus->ops->remove_bus()`
6. 架构特定总线移除：`pcibios_remove_bus()`
7. 注销总线设备：`device_unregister()`

### 6. 总线设备停止

#### pci_stop_bus_device
递归停止总线上的所有设备：

```c
static void pci_stop_bus_device(struct pci_dev *dev)
```

**处理逻辑：**
1. 先停止子总线（递归）
2. 再停止当前设备

#### pci_stop_bus
停止总线及其子总线的所有设备：

```c
static void pci_stop_bus(struct pci_bus *bus)
```

**遍历顺序：**
- 反向遍历设备（确保子总线先停止）

### 7. 设备移除接口

#### pci_stop_and_remove_bus_device
停止并移除总线设备：

```c
static void pci_stop_and_remove_bus_device(struct pci_dev *dev)
```

**完整移除：**
1. 先停止子总线
2. 停止当前设备
3. 移除子总线（递归）
4. 销毁当前设备

#### pci_stop_and_remove_bus
停止并移除总线和所有设备：

```c
static void pci_stop_and_remove_bus(struct pci_bus *bus)
```

**总线移除：**
1. 停止所有设备（反向遍历）
2. 移除所有设备（反向遍历）
3. 移除总线本身

### 8. 设备删除接口

#### pci_del_device
从总线删除设备：

```c
void pci_del_device(struct pci_dev *dev)
```

**删除流程：**
1. 停止设备
2. 销毁设备

## 设备生命周期

### 完整生命周期
```
创建 → 添加 → 探测 → 绑定 → 运行 → 移除 → 销毁
```

### 移除阶段
```
1. 停止阶段
   - 禁用PME
   - 释放驱动
   - 清理sysfs/proc

2. 销毁阶段
   - 删除kobject
   - 从总线移除
   - 释放资源
```

### 停止与销毁区别
- **停止**: 逻辑停止，设备结构仍存在
- **销毁**: 完全释放资源，设备结构将被释放

## 关键要点

1. **停止顺序**: 先子后父（反向遍历）
2. **锁保护**: 使用 `pci_bus_sem` 写锁保护总线列表
3. **资源清理**: 必须释放所有已声明的资源
4. **驱动释放**: 在停止阶段释放驱动引用
5. **kobject删除**: 在销毁前调用 `device_del()`
6. **清理顺序**: sysfs → proc → DT → kobject → 资源
7. **桥接处理**: 子总线必须先于父总线停止
