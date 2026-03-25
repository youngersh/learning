# PCI配置空间访问详细流程分析 (access.c)

## 整体架构

PCI配置空间访问提供多层API，从最底层的硬件访问到最高层的设备接口：

```
┌─────────────────────────────────────────────────────────────┐
│                     应用/驱动层                              │
│              pci_read/write_config_*()                      │
│              pcie_capability_*()                            │
├─────────────────────────────────────────────────────────────┤
│                     设备封装层                               │
│            pci_user_read/write_config_*()                   │
├─────────────────────────────────────────────────────────────┤
│                     总线访问层                               │
│          pci_bus_read/write_config_*()                      │
│          pci_generic_config_*()                             │
├─────────────────────────────────────────────────────────────┤
│                    硬件抽象层 (pci_ops)                      │
│          ops->read/write()                                  │
│          ops->map_bus()                                     │
└─────────────────────────────────────────────────────────────┘
         (平台特定实现)
```

## 流程一：标准配置空间读取

### pci_read_config_byte/word/dword()

**调用路径**：驱动/应用层 → 设备检查 → 总线访问

**详细执行流程**：

```
调用：pci_read_config_byte(dev, where, &value)
    ↓
1. 检查设备状态
    if (pci_dev_is_disconnected(dev)) {
        *value = 0xFF;           // 错误响应值
        return PCIBIOS_DEVICE_NOT_FOUND;
    }
    ↓
2. 转线到总线访问
    pci_bus_read_config_byte(dev->bus, dev->devfn, where, &value)
    ↓
3. 地址对齐检查（宏 PCI_byte_BAD）
    #define PCI_byte_BAD 0
    // byte访问：任意地址都有效
    ↓
4. 获取pci_lock锁
    raw_spin_lock_irqsave(&pci_lock, flags)
    // 除非 CONFIG_PCI_LOCKLESS_CONFIG
    ↓
5. 调用平台特定的read函数
    dev->bus->ops->read(bus, devfn, pos, 1, &data)
    // 例如：pci_conf1_read()
    ↓
6. 处理返回结果
    if (res != PCIBIOS_SUCCESSFUL) {
        *value = 0xFF;           // 错误响应
    } else {
        *value = (u8)data;
    }
    ↓
7. 释放锁
    raw_spin_unlock_irqrestore(&pci_lock, flags)
    ↓
8. 返回结果
    return PCIBIOS_SUCCESSFUL;
```

### pci_read_config_word()

**特殊处理 - 2字节对齐**：

```
调用：pci_read_config_word(dev, where, &value)
    ↓
1. 检查设备状态
    ↓
2. 地址对齐检查
    #define PCI_word_BAD (pos & 1)
    // 奇数地址：where % 2 != 0 时返回错误
    if (where & 1) {
        return PCIBIOS_BAD_REGISTER_NUMBER;
    }
    ↓
3. 获取锁
    raw_spin_lock_irqsave(&pci_lock, flags)
    ↓
4. 调用read函数
    bus->ops->read(bus, devfn, pos, 2, &data)
    // size = 2 (16-bit)
    ↓
5. 处理结果
    if (res) *value = 0xFFFF;
    else *value = (u16)data;
    ↓
6. 释放锁
    raw_spin_unlock_irqrestore(&pci_lock, flags)
```

### pci_read_config_dword()

**特殊处理 - 4字节对齐**：

```
调用：pci_read_config_dword(dev, where, &value)
    ↓
1. 检查设备状态
    ↓
2. 地址对齐检查
    #define PCI_dword_BAD (pos & 3)
    // 双字地址：where % 4 != 0 时返回错误
    if (where & 3) {
        return PCIBIOS_BAD_REGISTER_NUMBER;
    }
    ↓
3. 获取锁
    raw_spin_lock_irqsave(&pci_lock, flags)
    ↓
4. 调用read函数
    bus->ops->read(bus, devfn, pos, 4, &data)
    // size = 4 (32-bit)
    ↓
5. 处理结果
    if (res) *value = 0xFFFFFFFF;
    else *value = data;
    ↓
6. 释放锁
```

## 流程二：标准配置空间写入

### pci_write_config_byte/word/dword()

**写入流程**：

```
调用：pci_write_config_byte(dev, where, value)
    ↓
1. 检查设备断开状态
    if (pci_dev_is_disconnected(dev)) {
        return PCIBIOS_DEVICE_NOT_FOUND;
    }
    ↓
2. 转线到总线写入
    pci_bus_write_config_byte(dev->bus, dev->devfn, where, value)
    ↓
3. 地址对齐检查
    ↓
4. 获取pci_lock锁
    raw_spin_lock_irqsave(&pci_lock, flags)
    ↓
5. 调用平台特定的write函数
    bus->ops->write(bus, devfn, pos, 1, value)
    ↓
6. 释放锁
    raw_spin_unlock_irqrestore(&pci_lock, flags)
    ↓
7. 返回结果
```

## 流程三：用户空间配置访问

### 用户读取流程

**场景**：sysfs接口或用户程序通过 /sys/bus/pci/devices/.../config 读取

```
调用：pci_user_read_config_byte(dev, where, &value)
    ↓
1. 地址对齐检查
    if (where & 1) {  // byte: 总是有效
        return -EINVAL;
    }
    ↓
2. 获取pci_lock（原始自旋锁）
    raw_spin_lock_irq(&pci_lock)
    ↓
3. 检查配置访问是否被阻塞
    if (dev->block_cfg_access) {
        // 设备正在进行BIST或D状态转换
        // 进入等待循环
        do {
            raw_spin_unlock_irq(&pci_lock);
            ↓
            // 睡眠等待解除阻塞
            wait_event(pci_cfg_wait, !dev->block_cfg_access);
            ↓
            // 重新获取锁检查条件
            raw_spin_lock_irq(&pci_lock);
        } while (dev->block_cfg_access);
    }
    ↓
4. 调用read函数
    ret = dev->bus->ops->read(dev->bus, dev->devfn,
                                pos, sizeof(u8), &data);
    ↓
5. 释放锁
    raw_spin_unlock_irq(&pci_lock)
    ↓
6. 处理结果
    if (ret) {
        *value = 0xFF;  // 错误响应
    } else {
        *value = (u8)data;
    }
    ↓
7. 转换为errno返回
    return pcibios_err_to_errno(ret);
```

### 用户写入流程

```
调用：pci_user_write_config_byte(dev, where, value)
    ↓
1. (同读取的步骤1-3：对齐检查 + 阻塞检查)
    ↓
4. 调用write函数
    ret = dev->bus->ops->write(dev->bus, dev->devfn,
                                 pos, sizeof(u8), value);
    ↓
5. 释放锁
    raw_spin_unlock_irq(&pci_lock)
    ↓
6. 返回errno
    return pcibios_err_to_errno(ret);
```

## 流程四：配置访问锁定机制

### pci_cfg_access_lock()

**用途**：驱动需要在安全期间（如BIST、电源转换）阻塞用户访问

```
调用：pci_cfg_access_lock(dev)
    ↓
1. 确保可以在睡眠中调用
    might_sleep();
    ↓
2. 获取pci_lock
    raw_spin_lock_irq(&pci_lock)
    ↓
3. 检查并等待当前访问
    if (dev->block_cfg_access) {
        do {
            raw_spin_unlock_irq(&pci_lock);
            wait_event(pci_cfg_wait, !dev->block_cfg_access);
            raw_spin_lock_irq(&pci_lock);
        } while (dev->block_cfg_access);
    }
    ↓
4. 设置阻塞标志
    dev->block_cfg_access = 1;
    ↓
5. 释放锁
    raw_spin_unlock_irqrestore(&pci_lock, flags)
```

### pci_cfg_access_unlock()

**用途**：恢复配置访问

```
调用：pci_cfg_access_unlock(dev)
    ↓
1. 获取pci_lock
    raw_spin_lock_irqsave(&pci_lock, flags)
    ↓
2. 确保访问是被阻塞的
    WARN_ON(!dev->block_cfg_access);
    ↓
3. 清除阻塞标志
    dev->block_cfg_access = 0;
    ↓
4. 释放锁
    raw_spin_unlock_irqrestore(&pci_lock, flags)
    ↓
5. 唤醒所有等待的访问者
    wake_up_all(&pci_cfg_wait);
```

**并发场景**：
```
时间线：
T1: 驱动调用 pci_cfg_access_lock()
    - 设置 block_cfg_access = 1
T2: 用户尝试读取配置空间
    - 检测到 block_cfg_access = 1
    - 进入 wait_event() 等待
T3: 驱动调用 pci_cfg_access_unlock()
    - 设置 block_cfg_access = 0
    - wake_up_all() 唤醒T2
T2: 被唤醒，继续读取
```

### pci_cfg_access_trylock()

**用途**：原子上下文中尝试锁定（不能睡眠）

```
调用：pci_cfg_access_trylock(dev)
    ↓
1. 获取pci_lock
    raw_spin_lock_irqsave(&pci_lock, flags)
    ↓
2. 检查状态
    if (dev->block_cfg_access) {
        locked = false;  // 已被锁定
    } else {
        dev->block_cfg_access = 1;
        locked = true;  // 成功锁定
    }
    ↓
3. 释放锁
    raw_spin_unlock_irqrestore(&pci_lock, flags)
    ↓
4. 返回结果
    return locked;
```

## 流程五：：PCIe能力寄存器访问

### pcie_capability_read_word()

**场景**：读取PCIe扩展能力寄存器

```
调用：pcie_capability_read_word(dev, PCI_EXP_LNKCTL, &value)
    ↓
1. 地址对齐检查
    if (pos & 1) {  // word需要2字节对齐
        return PCIBIOS_BAD_REGISTER_NUMBER;
    }
    ↓
2. 检查能力是否实现
    if (!pcie_capability_reg_implemented(dev, pos)) {
        *value = 0;
        return 0;  // 成功但值为0
    }
    ↓
3. 能力实现检查详情
    switch (pos) {
        case PCI_EXP_LNKCTL:
            // 检查设备类型是否支持Link Control
            return pcie_cap_has_lnkctl(dev);
            // 支持类型：Endpoint, Root Port, Upstream,
            //           Downstream, PCIe Bridge
    }
    ↓
4. 读取配置寄存器
    ret = pci_read_config_word(dev, pci_pcie_cap(dev) + pos, &value);
    // pci_pcie_cap(dev) 返回PCIe Capability寄存器偏移
    ↓
5. 处理读取失败
    if (ret) {
        *value = 0;  // 失败时返回0
    }
    ↓
6. 特殊情况：Downstream Port的Slot Status
    if (pci_is_pcie(dev) && pcie_downstream_port(dev) &&
        pos == PCI_EXP_SLTSTA) {
        *value = PCI_EXP_SLTSTA_PDS;  // Presence Detect State
        return 0;
    }
    ↓
7. 返回结果
    return ret;
```

### pcie_capability_write_word()

```
调用：pcie_capability_write_word(dev, PCI_EXP_LNKCTL, value)
    ↓
1. 地址对齐检查
    ↓
2. 检查能力是否实现
    if (!pcie_capability_reg_implemented(dev, pos)) {
        return 0;  // 不实现，直接返回成功
    }
    ↓
3. 写入配置寄存器
    return pci_write_config_word(dev, pci_pcie_cap(dev) + pos, value);
```

### pcie_capability_clear_and_set_word_unlocked()

**用途**：原子修改PCIe能力寄存器（位操作）

```
调用：pcie_capability_clear_and_set_word_unlocked(dev, pos, clear, set)
    ↓
示例：清除并设置链路控制寄存器
    // 清除: PCI_EXP_LNKCTL_ASPM_L0S
    // 设置: PCI_EXP_LNKCTL_ASPM_L1
    ↓
1. 读取当前值
    ret = pcie_capability_read_word(dev, pos, &val);
    if (ret) {
        return ret;
    }
    ↓
2. 修改值
    val &= ~clear;   // 清除要清除的位
    val |= set;     // 设置要设置的位
    ↓
3. 写入新值
    return pcie_capability_write_word(dev, pos, val);
```

### pcie_capability_clear_and_set_word_locked()

**用途**：带设备锁的原子修改

```
调用：pcie_capability_clear_and_set_word_locked(dev, pos, clear, set)
    ↓
1. 获取设备级锁
    spin_lock_irqsave(&dev->pcie_cap_lock, flags);
    ↓
2. 执行解锁版本的操作
    ret = pcie_capability_clear_and_set_word_unlocked(dev, pos, clear, set);
    ↓
3. 释放设备级锁
    spin_unlock_irqrestore(&dev->pcie_cap_lock, flags);
    ↓
4. 返回结果
    return ret;
```

## 流程六：通用配置访问

### pci_generic_config_read()

**用途**：标准内存映射配置空间访问

```
调用：pci_generic_config_read(bus, devfn, where, size, &val)
    ↓
1. 映射配置空间地址
    addr = bus->ops->map_bus(bus, devfn, where);
    // map_bus返回io映射的虚拟地址
    if (!addr) {
        return PCIBIOS_DEVICE_NOT_FOUND;
    }
    ↓
2. 根据大小读取
    if (size == 1) {
        *val = readb(addr);     // 读8位
    } else if (size == 2) {
        *val = readw(addr);     // 读16位
    } else {
        *val = readl(addr);     // 读32位
    }
    ↓
3. 返回成功
    return PCIBIOS_SUCCESSFUL;
```

### pci_generic_config_write()

**用途**：标准内存映射写入

```
调用：pci_generic_config_write(bus, devfn, where, size, val)
    ↓
1. 映射配置空间地址
    addr = bus->ops->map_bus(bus, devfn, where);
    if (!addr) {
        return PCIBIOS_DEVICE_NOT_FOUND;
    }
    ↓
2. 根据大小写入
    if (size == 1) {
        writeb(val, addr);
    } else if (size == 2) {
        writew(val, addr);
    } else {
        writel(val, addr);
    }
    ↓
3. 返回成功
    return PCIBIOS_SUCCESSFUL;
```

### pci_generic_config_read32() - 仅32位访问硬件

**用途**：某些硬件仅支持32位配置访问

```
调用：pci_generic_config_read32(bus, devfn, where, size, &val)
    ↓
1. 映射配置空间地址
    // 注意：地址按4字节对齐 (& ~0x3)
    addr = bus->ops->map_bus(bus, devfn32, where & ~0x3);
    if (!addr) {
        return PCIBIOS_DEVICE_NOT_FOUND;
    }
    ↓
2. 读取32位数据
    *val = readl(addr);
    ↓
3. 提取所需部分
    if (size <= 2) {
        // 需要提取8位或16位
        shift = 8 * (where & 3);  // 0, 8, 16, 24
        mask = ((1 << (size * 8)) - 1);  // 0xFF, 0xFFFF
        *val = (*val >> shift) & mask;
    }
    // size == 4时，直接使用32位值
    ↓
4. 返回成功
    return PCIBIOS_SUCCESSFUL;
```

**示例**：
```
读取8位（奇数地址）：
  where = 0x13, 读取32位后右移8位
  *val = (readl(0x10) >> 8) & 0xFF;

读取16位（2字节对齐但非4字节）：
  where = 0x12, 读取32位后右移16位
  *val = (readl(0x10) >> 16) & 0xFFFF;
```

### pci_generic_config_write32() - 仅32位写入硬件

**用途**：处理只支持32位写入的硬件

```
调用：pci_generic_config_write32(bus, devfn, where, size, val)
    ↓
1. 检查size == 4的情况
    if (size == 4) {
        // 直接32位写入
        writel(val, addr);
        return PCIBIOS_SUCCESSFUL;
    }
    ↓
2. 发出警告（仅第一次）
    if (!bus->unsafe_warn) {
        dev_warn("32-bit hardware: 8/16-bit write may corrupt RW1C bits");
        bus->unsafe_warn = 1;
    }
    ↓
3. 读取-修改-写操作
    shift = (where & 3) * 8;           // 0, 8, 16, 24
    mask = ~(((1 << (size * 8)) - 1) << shift;
         // 例如size=1: ~0xFF << shift
    ↓
4. 读取当前32位值
    tmp = readl(addr);
    ↓
5. 清除要修改的位
    tmp = tmp & mask;
    ↓
6. 设置新的值
    tmp |= val << shift;
    ↓
7. 写入新值
    writel(tmp, addr);
    ↓
8. 返回成功
    return PCIBIOS_SUCCESSFUL;
```

**为什么需要读-修改-写**：
- RW1C位（Write-One-to-Clear）写1会清零
- 只修改要改变的位，保留其他位不变

## 流程七：操作函数动态设置

### pci_bus_set_ops()

**用途**：运行时切换配置访问实现

```
调用：pci_bus_set_ops(bus, new_ops)
    ↓
1. 获取pci_lock
    raw_spin_lock_irqsave(&pci_lock, flags);
    ↓
2. 保存旧操作
    old_ops = bus->ops;
    ↓
3. 设置新操作
    bus->ops = new_ops;
    ↓
4. 释放锁
    raw_spin_unlock_irqrestore(&pci_lock, flags);
    ↓
5. 返回旧操作
    return old_ops;
```

## 关键数据结构和锁

### pci_lock
```c
DEFINE_RAW_SPINLOCK(pci_lock);
```
- 原始自旋锁（raw_spinlock_t）
- 保护所有配置空间访问
- 中断安全（irqsafe）
- 在 `CONFIG_PCI_LOCKLESS_CONFIG` 时禁用

### block_cfg_access
```c
struct pci_dev {
    ...
    int block_cfg_access;  // 配置访问阻塞标志
    ...
};
```
- 每设备一个标志
- 驱动在BIST或D状态转换时设置
- 阻塞用户空间访问

### pci_cfg_wait
```c
static DECLARE_WAIT_QUEUE_HEAD(pci_cfg_wait);
```
- 等待队列
- 用户访问被阻塞时在此等待

## 完整访问场景示例

### 场景1：驱动读取设备BAR

```
1. 驱动probe函数
2. 调用 pci_read_config_dword(dev, PCI_BASE_ADDRESS_0, &bar0)
   → 检查设备断开状态
   → 地址对齐检查（where % 4 == 0）
   → 获取pci_lock
   → 调用 bus->ops->read()
   → 释放pci_lock
3. bar0包含设备BAR地址
```

### 场景2：用户通过sysfs写入配置

```
1. 用户执行：echo 0x1234 > /sys/.../config
2. 内核调用 pci_user_write_config_word()
   → 对齐检查
   → 获取pci_lock
   → 检查 block_cfg_access（可能被BIST阻塞）
   → 如果阻塞，等待唤醒
   → 调用 bus->ops->write()
   → 释放pci_lock
3. 返回给用户
```

### 场景3：驱动执行BIST测试

```
1. 驱动开始BIST测试
2. 调用 pci_cfg_access_lock(dev)
   → 设置 dev->block_cfg_access = 1
   → 唤醒等待的用户访问（如果有）
3. 执行BIST操作
4. 调用 pci_cfg_access_unlock(dev)
   → 清除 dev->block_cfg_access = 0
   → 唤醒所有等待者
```

### 场景4：PCIe能力修改

```
1. 驱动需要修改链路控制寄存器
2. 调用 pcie_capability_clear_and_set_word_locked()
   → 获取 dev->pcie_cap_lock
   → 读取当前值
   → val &= ~PCI_EXP_LNKCTL_ASPM_L0S
   → val |= PCI_EXP_LNKCTL_ASPM_L1
   → 写入新值
   → 释放 dev->pcie_cap_lock
```

## 总结

### 访问层次
1. **应用层**：`pci_read/write_config_*` - 设备接口
2. **用户层**：`pci_user_read/write_config_*` - sysfs接口
3. **总线层**：`pci_bus_read/write_config_*` - 总线接口
4. **硬件层**：`bus->ops->read/write` - 平台实现

### 保护机制
1. **pci_lock**：全局锁，保护所有配置访问
2. **block_cfg_access**：设备标志，阻塞用户访问
3. **等待队列**：阻塞时让用户等待

### 特殊处理
1. **对齐检查**：word需要2字节对齐，dword需要4字节对齐
2. **错误响应**：失败时返回0xFF/0xFFFF/0xFFFFFFFF
3. **32位硬件**：部分硬件只支持32位访问
4. **PCIe能力验证**：检查能力是否在设备上实现
