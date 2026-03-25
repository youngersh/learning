# Vital Product Data (VPD) Capability 详细分析

## 概述

**Capability ID**: PCI_CAP_ID_VPD (0x03)  
**规范参考**: PCI VPD Specification  
**功能**: 存储设备的 vital product data 信息

## PCIe SPEC 定义

### Capability 结构
```
Offset  Size  Description
0x00    1    Capability ID (0x03)
0x01    1    Next Capability Pointer
0x02    2    VPD Data Register (32-bit)
0x04    2    VPD Address Register (16-bit)
              - Bits [14:0]: Address
              - Bit 15: Write Completion Flag
```

### VPD Tags
| Tag | 名称 | 描述 |
|-----|------|------|
| 0x82 | VPD-R | 只读数据 |
| 0x83 | VPD-W | 可写数据 |
| 0x90 | Large Resource Data | 大型资源数据 |
| 0x91 | Large Resource Data Tag | 大型资源数据标签 |
| 0x00 | End Tag | VPD 结束 |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/vpd.c`  
**函数**: `pci_vpd_init()` (行 264-271)

```c
void pci_vpd_init(struct pci_dev *dev)
{
    // 1. 检查 VPD 长度
    if (dev->vpd.len == PCI_VPD_SZ_INVALID)
        return;

    // 2. 查找 VPD capability
    dev->vpd.cap = pci_find_capability(dev, PCI_CAP_ID_VPD);

    // 3. 初始化互斥锁
    mutex_init(&dev->vpd.lock);
}
```

### 读取 VPD

**函数**: `pci_read_vpd()` (内部函数)

```c
int pci_read_vpd(struct pci_dev *dev, loff_t pos, size_t count,
                 void *buf)
{
    struct vpd_priv *vpd = &dev->vpd;
    loff_t end = pos + count;
    int ret = 0;
    u8 *vpd_data;
    bool go = vpd->valid;

    // 1. 检查 VPD 是否有效
    if (!go)
        return -EINVAL;

    // 2. 检查读取范围
    if (pos < 0 || pos > vpd->len || end > vpd->len)
        return -EINVAL;

    // 3. 检查是否在缓存中
    if (vpd->base) {
        // 从缓存读取
        memcpy(buf, vpd->base + pos, count);
        return count;
    }

    // 4. 获取 VPD 数据
    mutex_lock(&vpd->lock);
    vpd_data = vpd->base;

    // 5. 读取数据
    while (pos < end) {
        ret = pci_read_vpd_dword(dev, pos, vpd_data);
        if (ret < 0)
            break;

        pos += 4;
        vpd_data += 4;
    }

    mutex_unlock(&vpd->lock);
    return ret;
}
```

### 写入 VPD

**函数**: `pci_write_vpd()` (内部函数)

```c
int pci_write_vpd(struct pci_dev *dev, loff_t pos, size_t count,
                  const void *buf)
{
    struct vpd_priv *vpd = &dev->vpd;
    loff_t end = pos + count;
    int ret = 0;

    // 1. 检查 VPD 是否有效
    if (!vpd->valid)
        return -EINVAL;

    // 2. 检查写入范围
    if (pos < 0 || pos > vpd->len || end > vpd->len)
        return -EINVAL;

    // 3. 检查是否可写
    if (!vpd->writable)
        return -EPERM;

    // 4. 写入数据
    mutex_lock(&vpd->lock);

    while (pos < end) {
        ret = pci_write_vpd_dword(dev, pos, buf);
        if (ret < 0)
            break;

        pos += 4;
        buf += 4;
    }

    mutex_unlock(&vpd->lock);
    return ret;
}
```

### VPD 缓存

**函数**: `pci_vpd_alloc()` (行 343-369)

```c
void *pci_vpd_alloc(struct pci_dev *dev, unsigned int *size)
{
    unsigned int len;
    void *buf;
    int cnt;

    // 1. 检查 VPD 是否可用
    if (!pci_vpd_available(dev, true))
        return ERR_PTR(-ENODEV);

    // 2. 获取 VPD 长度
    len = dev->vpd.len;

    // 3. 分配缓冲区
    buf = kmalloc(len, GFP_KERNEL);
    if (!buf)
        return ERR_PTR(-ENOMEM);

    // 4. 读取 VPD 数据

 cnt = pci_read_vpd(dev, 0, len, buf);
    if (cnt != len) {
        kfree(buf);
        return ERR_PTR(-EIO);
    }

    *size = len;
    return buf;
}
```

## Root Port 处理

Root Port 通常不实现 VPD capability，因为它是 Bridge 设备。VPD 主要用于 Endpoint 设备。

## 设备驱动使用

### 1. 读取 VPD 数据
```c
// 分配 VPD 缓冲区
unsigned int vpd_size;
void *vpd = pci_vpd_alloc(pdev, &vpd_size);
if (IS_ERR(vpd)) {
    dev_err(&pdev->dev, "Failed to allocate VPD: %ld\n",
             PTR_ERR(vpd));
    return PTR_ERR(vpd);
}

// 使用 VPD 数据
// ...

// 释放 VPD
kfree(vpd);
```

### 2. 检查 VPD 支持
```c
if (pdev->vpd.cap) {
    // 设备支持 VPD
    
    // 检查 VPD 是否有效
    if (pdev->vpd.valid) {
        dev_info(&pdev->dev, "VPD is valid, size: %d\n",
                 pdev->vpd.len);
    }
}
```

### 3. sysfs 接口
```c
// VPD 通过 sysfs 暴露
// /sys/bus/pci/devices/xxxx:xx:xx.x/vpd

// 读取 VPD
cat /sys/bus/pci/devices/0000:01:00.0/vpd > vpd.bin

// 写入 VPD（如果可写）
echo -n data > /sys/bus/pci/devices/0000:01:00.0/vpd
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
()
       └─> pci_init_capabilities()
            └─> pci_vpd_init()
                 ├─> pci_find_capability(PCI_CAP_ID_VPD)
                 └─> mutex_init()  // 初始化互斥锁

驱动调用:
  └─> pci_vpd_alloc()
       ├─> pci_read_vpd()  // 读取 VPD 数据
       └─> kmalloc()  // 分配缓冲区
```

## 配置空间访问

### 查找 VPD Capability
```c
int vpd_pos = pci_find_capability(dev, PCI_CAP_ID_VPD);
```

### 读取 VPD Address
```c
u16 addr;
pci_read_config_word(dev, vpd_pos + PCI_VPD_ADDR, &addr);

// 检查写完成标志
bool write_complete = (addr & PCI_VPD_ADDR_F) != 0;

// 获取地址
u16 vpd_addr = addr & PCI_VPD_ADDR_MASK;
```

### 读取 VPD Data
```c
u32 data;
pci_read_config_dword(dev, vpd_pos + PCI_VPD_DATA, &data);
```

### 写入 VPD Address
```c
// 设置地址
pci_write_config_word(dev, vpd_pos + PCI_VPD_ADDR, vpd_addr);

// 等待写完成
usleep_range(10, 1000);  // 10-1000us

// 检查写完成标志
pci_read_config_word(dev, vpd_pos + PCI_VPD_ADDR, &addr);
if (addr & PCI_VPD_ADDR_F) {
    // 写完成
}
```

## VPD 读取流程

```
1. 设置 VPD Address
    ↓
2. 等待写完成
    ↓
3. 读取 VPD Data
    ↓
4. 重复直到读取完所有数据
```

## VPD 标签解析

```
VPD 数据格式:
├─ Tag (2 bytes)
├─ Length (2 bytes)
└─ Data (Length bytes)

特殊标签:
- 0x82: VPD-R (只读数据)
- 0x83: VPD-W (可写数据)
- 0x90/0x91: 大型资源数据
- 0x00: 结束标签
```

## 性能考虑

1. **读取延迟**: VPD 读取需要等待写完成
2. **缓存**: VPD 数据可以缓存到内存中
3. **并发访问**: 使用互斥锁保护
4. **大小限制**: VPD 大小通常限制在 32KB

## 调试信息

```c
// VPD 读取错误
pci_err(dev, "VPD access failed: %d\n", ret);

// VPD 无效
dev->vpd.len = PCI_VPD_SZ_INVALID;
```

## 错误处理

1. **VPD 不支持**:
   ```c
   if (!dev->vpd.cap)
       return;
   ```

2. **VPD 无效**:
   ```c
   if (dev->vpd.len == PCI_VPD_SZ_INVALID)
       return;
   ```

3. **读取失败**:
   ```c
   ret = pci_read_vpd(dev, pos, count, buf);
   if (ret < 0) {
       dev_err(&dev->dev, "VPD read failed: %d\n", ret);
       return ret;
   }
   ```

4. **写入失败**:
   ```c
   ret = pci_write_vpd(dev, pos, count, buf);
   if (ret < 0) {
       dev_err(&dev->dev, "VPD write failed: %d\n", ret);
       return ret;
   }
   ```

## 相关配置选项

- `CONFIG_PCI`: 启用 PCI 支持
- `CONFIG_SYSFS`: sysfs 接口支持

## 参考资料

- PCI VPD Specification
- PCI Local Bus Specification 3.0+, Section 6.7
- Linux 内核源码: `drivers/pci/vpd.c`
- 头文件: `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
