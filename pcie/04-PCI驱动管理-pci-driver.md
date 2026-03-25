# PCI驱动管理分析 (pci-driver.c)

## 文件概述
`pci-driver.c` 实现PCI驱动的注册、匹配、探测和移除功能，是PCI驱动模型的核心。

## 核心数据结构

### pci_dynid
```c
struct pci_dynid {
    struct list_head node;
    struct pci_device_id id;
};
```
- 动态添加的设备ID，通过sysfs接口管理

## 主要功能函数

### 1. 驱动注册与注销

#### __pci_register_driver
注册PCI驱动：

```c
int __pci_register_driver(struct pci_driver *drv, struct module *owner,
                      const char *mod_name)
```

**初始化内容：**
- 设置驱动名称、总线类型
- 设置模块所有者和名称
- 初始化动态ID列表（spinlock + list）
- 调用 `driver_register()` 注册到驱动核心

#### pci_unregister_driver
注销PCI驱动：

```c
void pci_unregister_driver(struct pci_driver *drv)
```

**清理流程：**
- 调用 `driver_unregister()`
- 释放所有动态ID

### 2. 设备ID匹配

#### pci_match_id
匹配静态设备ID表：

```c
const struct pci_device_id *pci_match_id(const struct pci_device_id *ids,
                                      struct pci_dev *dev)
```

**匹配规则：**
- 厂商ID匹配
- 设备ID匹配
- 子厂商/设备ID匹配
- 类码匹配

#### pci_match_device
完整设备匹配（含动态ID）：

```c
static const struct pci_device_id *pci_match_device(struct pci_driver *drv,
                                             struct pci_dev *dev)
```

**匹配优先级：**
1. 检查 `driver_override` 强制匹配
2. 先匹配动态ID（`drv->dynids`）
3. 再匹配静态ID表（`drv->id_table`）
4. 支持override_only标志

### 3. 动态ID管理

#### pci_add_dynid
通过sysfs添加设备ID：

```c
int pci_add_dynid(struct pci_driver *drv,
              unsigned int vendor, unsigned int device,
              unsigned int subvendor, unsigned int subdevice,
              unsigned int class, unsigned int class_mask,
              unsigned long driver_data)
```

**处理流程：**
1. 创建动态ID结构
2. 添加到驱动ID列表
3. 调用 `driver_attach()` 重新探测所有设备

#### new_id_store/remove_id_store
sysfs接口实现：

```c
static ssize_t new_id_store(struct device_driver *driver, const char *buf, size_t count)
static ssize_t remove_id_store(struct device_driver *driver, const char *buf, size_t count)
```

**格式：** `vendor device subvendor subdevice class class_mask driver_data`

### 4. 设备探测

#### pci_device_probe
设备探测入口点：

```c
static int pci_device_probe(struct device *dev)
```

**探测流程：**
1. 检查SR-IOV VF自动探测标志
2. 分配IRQ: `pci_assign_irq()`
3. 调用架构特定 `pcibios_alloc_irq()`
4. 获取设备引用
5. 调用 `__pci_device_probe()` 执行实际探测

#### __pci_device_probe
探测驱动：

```c
static int __pci_device_probe(struct pci_driver *drv, struct pci_dev *pci_dev)
```

**探测步骤：**
1. 匹配设备ID
2. 调用 `pci_call_probe()` 执行驱动的probe函数

#### pci_call_probe
在特定CPU上执行probe：

```c
static int pci_call_probe(struct pci_driver *drv, struct pci_dev *dev,
                         const struct pci_device_id *id)
```

**NUMA绑定逻辑：**
- 获取设备所属NUMA节点
- 使用 `work_on_cpu()` 在该节点执行probe
- 目的：驱动在本地分配内存

#### local_pci_probe
实际的probe执行函数：

```c
static long local_pci_probe(void *_ddi)
```

**探测过程：**
1. 获取runtime PM引用
2. 设置 `pci_dev->driver`
3. 调用驱动probe函数
4. 处理返回值（0=成功，<0=失败，>0=警告）

### 5. 设备移除

#### pci_device_remove
设备移除入口点：

```c
static void pci_device_remove(struct device *dev)
```

**移除流程：**
1. 获取PM同步引用
2. 等待runtime PM完成
3. 调用驱动remove函数
4. 释放IRQ: `pcibios_free_irq()`
5. 清除驱动指针
6. 处理SR-IOV
7. 释放runtime PM引用

### 6. 设备关闭

#### pci_device_shutdown
设备关闭（kexec时调用）：

```c
static void pci_device_shutdown(struct device *dev)
```

**关闭操作：**
1. 恢复到D0状态
2. 调用驱动shutdown函数
3. kexec时清除Bus Master位

### 7. 电源管理

#### pci_pm_ops
PCI设备电源操作结构：

```c
static const struct dev_pm_ops pci_dev_pm_ops = {
    .prepare = pci_pm_prepare,
    .complete = pci_pm_complete,
    .suspend = pci_pm_suspend,
    .suspend_late = pci_pm_suspend_late,
    .resume = pci_pm_resume,
    .resume_early = pci_pm_resume_early,
    .freeze = pci_pm_freeze,
    .thaw = pci_pm_thaw,
    .poweroff = pci_pm_poweroff,
    .poweroff_late = pci_pm_poweroff_late,
    .restore = pci_pm_restore,
    .suspend_noirq = pci_pm_suspend_noirq,
    .resume_noirq = pci_pm_resume_noirq,
    .freeze_noirq = pci_pm_freeze_noirq,
    .thaw_noirq = pci_pm_thaw_noirq,
    .poweroff_noirq = pci_pm_poweroff_noirq,
    .restore_noirq = pci_pm_restore_noirq,
    .runtime_suspend = pci_pm_runtime_suspend,
    .runtime_resume = pci_pm_runtime_resume,
    .runtime_idle = pci_pm_runtime_idle,
};
```

**PM回调处理：**
- **suspend阶段**: 保存配置，进入低功耗状态
- **resume阶段**: 恢复配置，重新使能设备
- **runtime PM**: 支持设备运行时电源管理
- **legacy PM**: 支持旧式suspend/resume回调

### 8. 总线类型注册

#### pci_bus_type
PCI总线类型定义：

```c
const struct bus_type pci_bus_type = {
    .name = "pci",
    .match = pci_bus_match,
    .uevent = pci_uevent,
    .probe = pci_device_probe,
    .remove = pci_device_remove,
    .shutdown = pci_device_shutdown,
    .irq_get_affinity = pci_device_irq_get_affinity,
    .dev_groups = pci_dev_groups,
    .bus_groups = pci_bus_groups,
    .drv_groups = pci_drv_groups,
    .pm = PCI_PM_OPS_PTR,
    .num_vf = pci_bus_num_vf,
            .dma_configure = pci_dma_configure,
            .dma_cleanup = pci_dma_cleanup,
};
```

## 重要设计模式

### 1. 驱动与设备绑定
```c
struct pci_driver {
    char *name;
    const struct pci_device_id *id_table;
    int (*probe)(struct pci_dev *dev, const struct pci_device_id *id);
    void (*remove)(struct pci_dev *dev);
    // ...
};
```

### 2. 设备ID结构
```c
struct pci_device_id {
    __u32 vendor;
    __u32 device;
    __u32 subvendor;
    __u32 subdevice;
    __u32 class;
    __u32 class_mask;
    kernel_ulong_t driver_data;
};
```

### 3. 驱动生命周期
1. 注册: `pci_register_driver()`
2. 探测: `probe()` 匹配并初始化
3. 运行: 设备正常工作
4. 移除: `remove()` 清理并断开

### 4. 动态ID支持
- 允许运行时添加/移除设备ID
- 通过sysfs `new_id` 和 `remove_id` 接口
- 触发重新探测所有设备

### 5. NUMA绑定
- probe在设备所属NUMA节点执行
- 优化内存访问局部性
- 使用workqueue机制

## 关键要点

1. **ID匹配优先级**: 动态ID > 静态ID
2. **原子性保证**: 使用spinlock保护动态ID列表
3. **NUMA优化**: probe在本地节点执行
4. **PM集成**: 完整的suspend/resume/runtime PM支持
5. **设备引用**: 探测/移除使用dev_get/put
6. **kexec处理**: 关闭时清除Bus Master防止DMA
