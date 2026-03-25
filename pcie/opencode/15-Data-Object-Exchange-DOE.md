# Data Object Exchange (DOE) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_DOE (0x2E)  
**规范参考**: PCIe DOE Specification  
**功能**: 提供安全数据对象交换协议

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x2E)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    4    DOE Capability
              - Bits [3:0]: DOE Supp Extended
              - Bits [11:4]: DOE Int Model
              - Bit 16: DOE BAR Offset
              - Bits [19:16: DOE Size Upper
              - Bit 31:20: DOE Size Lower
```

### DOE Int Model 值�
| 值 | 描述 |
|-----|------|
| 0x0 | 未指定 |
| 0x1 | Cacheable |
| 0x2 | Non-cacheable |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/doe.c`  
**函数**: `pci_doe_init()` (行 849-881)

```c
void pci_doe_init(struct pci_dev *pdev)
{
    struct pci_doe *doe;

    // 1. 检查是否是 Endpoint
    if (pci_pcie_type(pdev) != PCI_EXP_TYPE_ENDPOINT &&
        pci_pcie_type(pdev) != PCI_EXP_TYPE_RC_END &&
        pci_pcie_type(pdev) != PCI_EXP_TYPE_LEG_END)
        return;

    // 2. 分配 DOE 结构
    doe = kzalloc(sizeof(*doe), GFP_KERNEL);
    if (!doe)
        return;

    // 3. 初始化 DOE 结构
    doe->pdev = pdev;
    doe->cap_offset = 0;
    doe->wq = 1;
    INIT_DELAYED_WORK(&doe->wq, msecs_to_jiffies(10));

    // 4. 查找 DOE capability
    doe->cap_offset = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_DOE);
    if (!doe->cap_offset)
        goto out;

    // 5. 读取 DOE Capability
    pci_read_config_dword(pdev, doe->cap_offset + PCI_DOE_CAP, &doe->cap);

    // 6. 检查 DOE Int Model
    doe->intf = FIELD_GET(PCI_DOE_CAP_INT_MODEL, doe->cap);
    if (doe->intf == PCI_DOE_CAP_INT_MODEL_NOT_SUPP)
        goto out;

    // 7. 读取 BAR Offset
    doe->bar_offset = FIELD_GET(PCI_DOE_CAP_BAR_OFFSET, doe->cap);

    // 8. 读取 DOE Size
    doe->size_upper = FIELD_GET(PCI_DOE_CAP_SIZE_UPPER, doe->cap);
    doe->size_lower = FIELD_GET(PCI_DOE_CAP_SIZE_LOWER, doe->cap);
    doe->size = (doe->size_upper << 16) | doe->size_lower;

    // 9. 创建 DOE 对象队列
    doe->queue = kzalloc(sizeof(*doe->queue), GFP_KERNEL);
    if (!doe->queue)
        goto out;

    INIT_LIST_HEAD(&doe->queue);

    // 10. 添加到设备结构
    pdev->doe = doe;

    // 11. 创建 DOE 任务
    doe->task = kthread_create_worker(0, pci_doe_task, doe,
                                        "pcie_doe", pdev->dev);
    if (IS_ERR(doe->task))
        goto out;

    // 12. 启动 DOE 任务
    kthread_unpark(doe->task);
    return;

out:
    if (doe)
        kfree(doe->queue);
    kfree(doe);
}
```

### DOE 任务

**函数**: `pci_doe_task()` (行 783-847)

```c
static int pci_doe_task(void *data)
{
    struct pci_doe *doe = data;
    struct pci_dev *pdev = doe->pdev;
    struct pci_doe_mb *mb;
    int rc;

    // 1. 创建 DOE mailbox
    mb = pci_doe_create_mb(doe);
    if (!mb)
        return;

    // 2. 初始化 DOE mailbox
    rc = pci_doe_init_mb(doe, mb);
    if (rc)
        goto out;

    // 3. 添加 Data Object
    rc = pci_doe_add_do(doe, mb);
    if (rc)
        goto out;

    // 4. 发送 DOE Discovery
    rc = pci_doe_discovery(doe, mb);
    if (rc)
        goto out;

    // 5. 等待 DOE 初始化完成
    rc = wait_for_completion_timeout(&doe->wq,
                     msecs_to_jiffies(10));
    if (rc)
        goto out;

    // 6. 设置 DOE 为就绪
    pci_doe_set_ready(doe);

out:
    pci_doe_destroy(doe);
    return 0;
}
```

### 创建 DOE Mailbox

**函数**: `pci_doe_create_mb()` (行 695-732)

```c
static struct pci_doe_mb *pci_doe_create_mb(struct pci_doe *doe)
{
    struct pci_doe_mb *mb;
    u32 val;

    // 1. 分配 DOE mailbox 结构
    mb = kzalloc(sizeof(*mb), GFP_KERNEL);
    if (!mb)
        return NULL;

    // 2. 初始化 mailbox
    mb->doe = doe;
    mb->pdev = doe->pdev;
    INIT_LIST_HEAD(&mb->objs);

    // 3. 设置 DOE �存器属性
    mb->wq = 1;
    INIT_DELAYED_WORK(&mb->wq, msecs_to_jiffies(10));
    mb->timeout_jiffies = msecs_to_jiffies(60);
    mb->abort_timeout_jiffies = msecs_to_jiffies(10);

    // 4. 添加到对象列表
    list_add_tail(&doe->queue, mb);

    return mb;
}
```

### 添加 Data Object

**函数**: `pci_doe_add_do()` (行 734-782)

```c
static int pci_doe_add_do(struct pci_doe *doe, struct pci_doe_mb *mb)
{
    struct pci_doe_obj *obj;
    u32 val;

    // 1. 分配 DOE 对象
    obj = kzalloc(sizeof(*obj), GFP_KERNEL);
    if (!obj)
        return -ENOMEM;

    // 2. 初始化 DOE 对象
    obj->doe = doe;
    obj->mb = mb;
    obj->pdev = doe->pdev;
    obj->type = PCI_DOE_TYPE_DO;
    obj->doe_offset = doe->cap_offset;
    obj->data_obj = PCI_DOE_DATA_OBJ_DISC_REQ;

    // 3. 添加到对象列表
    list_add_tail(&mb->objs, obj);

    return 0;
}
```

### 发送 DOE Discovery

**函数**: `pci_doe_discovery()` (行 784-812)

```c
static int pci_doe_discovery(struct pci_doe *doe, struct pci_doe_mb *mb)
{
    struct pci_doe_obj *obj;
    u32 val;

    // 1. 创建 Discovery 对象
    obj = kzalloc(sizeof(*obj), GFP_KERNEL);
    if (!obj)
        return -ENOMEM;

    // 2. 初始化 Discovery 对象
    obj->doe = doe;
    obj->mb = mb;
    obj->pdev = doe->pdev;
    obj->type = PCI_DOE_TYPE_DISC_REQ;
    obj->doe_offset = doe->cap_offset;
    obj->data_obj = PCI_DOE_DATA_OBJ_DISC_REQ;

    // 3. 添加到对象列表
    list_add_tail(&mb->objs, obj);

    // 4. 发送 Discovery
    pci_doe_write_mbox(doe, mb);

    return 0;
}
```

### 设置 DOE 就绪

**函数**: `pci_doe_set_ready()` (行 814-823)

```c
void pci_doe_set_ready(struct pci_doe *doe)
{
    // 1. 设置 DOE 为就绪
    doe->ready = true;

    // 2. 通知 sysfs
    pci_doe_sysfs_init(doe->pdev);
}
```

### 销毁 DOE

**函数**: `pci_doe_destroy()` (行 825-848)

```c
void pci_doe_destroy(struct pci_doe *doe)
{
    struct pci_doe_mb *mb, *next;

    // 1. 停止 DOE 任务
    if (doe->task)
        kthread_stop_sync(doe->task);

    // 2. 销毁 DOE mailbox
    list_for_each_entry_safe(&doe->queue, mb, next) {
        pci_doe_destroy_mb(mb);
    }

    // 3. 释放 DOE 结构
    kfree(doe->queue);
    kfree(doe);

    // 4. 清除设备指针
    pdev->doe = NULL;
}
```

### DOE 断开连接

**函数**: `pci_doe_disconnected()` (行 597-603)

```c
void pci_doe_disconnected(struct pci_dev *dev)
{
    struct pci_doe *doe = dev->doe;

    if (!doe)
        return;

    // 1. 设置 DOE 为未就绪
    doe->ready = false;

    // 2. 停止 DOE 任务
    if (doe->task)
        kthread_stop_sync(doe->task);
}
```

## Root Port 处理

Root Port 通常不实现 DOE capability，因为它是 Bridge 设备。DOE 主要用于 Endpoint 设备。

## 设备驱动使用

### 1. 检查 DOE 支持
```c
if (pdev->doe) {
    // 设备支持 DOE
    
    if (pdev->doe->ready) {
        // DOE 已就绪
        dev_info(&pdev->dev, "DOE is ready\n");
    }
}
```

### 2. 发送 DOE 消息

```c
// 通过 sysfs 接口发送 DOE 消息
echo "message" > /sys/bus/pci/devices/xxxx:xx:xx.x/doe_discovery

// 读取 DOE 响应
cat /sys/bus/pci/devices/xxxx:xx:xx.x/doe_discovery
```

### 3. sysfs 接口

```c
// DOE sysfs 接口
/sys/bus/pci/devices/xxxx:xx:xx.x/doe_discovery  // 发送 Discovery
/sys/bus/pci/devices/xxxx:xx:xx.x/doe_data_obj     // 发送 Data Object
/sys/bus/pci/devices/xxxx:xx:xx.x/doe_config      // 配置 DOE
/sys/bus/pci/devices/xxxx:xx:xx.x/doe_status       // DOE 状态
```

## 调用流程

```
设备挂起:
  └─> pci_init_capabilities()
       └─> pci_doe_init()
            ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_DOE)
            ├─> pci_read_config_dword()  // 读取 DOE Capability
            ├─> kthread_create_worker()  // 创建 DOE 任务
            └─> pci_doe_task()  // 执行 DOE 任务
                 ├─> pci_doe_create_mb()  // 创建 DOE mailbox
                 ├─> pci_doe_init_mb()  // 初始化 mailbox
                 ├─> pci_doe_add_do()  // 添加 Data Object
                 └─> pci_doe_discovery()  // 发送 Discovery
                 └─> wait_for_completion_timeout()  // 等待完成
                 └─> pci_doe_set_ready()  // 设置就绪
```

DOE 消息处理:
  └─> sysfs 写入
  └─> pci_doe_write_mbox()
  └─> pci_doe_write_mbox()  // 写入配置空间
  └─> 设备处理 DOE 消息
```

## 配置空间访问

### 查找 DOE Capability
```c
int doe_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_DOE);
```

### 读取 DOE Capability
```c
u32 cap;
pci_read_config_dword(dev, doe_pos + PCI_DOE_CAP, &cap);

// 获取 DOE Int Model
u8 intf = FIELD_GET(PCI_DOE_CAP_INT_MODEL, cap);

// 获取 BAR Offset
u16 bar_offset = FIELD_GET(PCI_DOE_CAP_BAR_OFFSET, cap);

// 获取 DOE Size
u32 size_upper = FIELD_GET(PCI_DOE_CAP_SIZE_UPPER, cap);
u32 size_lower = FIELD_GET(PCI_DOE_CAP_SIZE_LOWER, cap);
u32 size = (size_upper << 16) | size_lower;
```

## DOE 消息格式

### Discovery 消息
```
{
    "doe_type": "Discovery",
    "doe_offset": capability_offset,
    "data_obj": "Discovery Request",
    "vendor_id": vendor_id,
    "data_length": 0,
    "data": []
}
```

### Data Object 消息
```
{
    "doe_type": "Data Object",
    "doe_offset": capability_offset,
    "data_obj": "Data Object Request",
    "vendor_id": vendor_id,
    "data_length": data_length,
    "data": [data_bytes...]
}
```

### DOE Config 消息
```
{
    "doe_type": "Configuration",
    "doe_offset": capability_offset,
    "data_obj": "Configuration Request",
    "vendor_id": vendor_id,
    "data_length": data_length,
    "data": [config_bytes...]
}
```

## 性能考虑

1. **异步初始化**: DOE 初始化是异步的
2. **超时处理**: DOE 操作有超时保护
3. **mailbox 管理**: 多个 Data Objects 共享一个 mailbox
4. **sysfs 接口**: 通过 sysfs 提供用户空间接口

## 调试信息

```c
// DOE 初始化
dev_info(&pdev->dev, "DOE capability detected\n");

// DOE 就绪
if (pdev->doe && pdev->doe->ready) {
    dev_info(&pdev->dev, "DOE is ready\n");
}
```

## 错误处理

1. **DOE 不支持**:
   ```c
   if (!doe->cap_offset)
       goto out;
   ```

2. **Int Model 不支持**:
   ```c
   if (doe->intf == PCI_DOE_CAP_INT_MODEL_NOT_SUPP)
       goto out;
   ```

3. **内存分配失败**:
   ```c
   doe = kzalloc(sizeof(*doe), GFP_KERNEL);
   if (!doe)
       return;
   ```

4. **创建任务失败**:
   ```c
   doe->task = kthread_create_worker(0, pci_doe_task, doe,
                                        "pcie_doe", pdev->dev);
   if (IS_ERR(doe->task))
       goto out;
   ```

## 相关配置选项

- `CONFIG_PCI_DOE`: 启用 DOE 支持
- `CONFIG_SYSFS`: sysfs 支持（DOE sysfs 接口的依赖）

## 参考资料

- PCIe Data Object Exchange Specification
- PCIe Base Specification Revision 5.0+, Section 6.30
- Linux 内核源码: `drivers/pci/doe.c`
- 头文件: `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
