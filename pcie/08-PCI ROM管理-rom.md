# PCI ROM管理分析 (rom.c)

## 文件概述
`rom.c` 实现PCI设备ROM（Option ROM）的访问和管理功能。

## ROM类型

### ROM BAR
- 标准PCI BAR之一，用于设备扩展ROM
- 可以映射到内存空间
- 支持使能/禁用解码

### ROM资源类型
1. **物理ROM**: 实际的设备ROM
2. **Shadow Copy**: RAM中的ROM复制（更快访问）

## 主要功能函数

### 1. ROM使能控制

#### pci_enable_rom
使能ROM解码：

```c
int pci_enable_rom(struct pci_dev *pdev)
```

**使能过程：**
1. 检查ROM BAR是否存在
2. 检查是否为shadow copy（shadow copy无需使能）
3. 读取ROM BAR当前值
4. 设置使能位（`PCI_ROM_ADDRESS_ENABLE`）
5. 写入新值到ROM BAR寄存器

**特殊处理：**
- 某些设备的ROM BAR在禁用时读为0
- 使能前不能简单设置位，必须保留地址信息

#### pci_disable_rom
禁用ROM解码：

```c
void pci_disable_rom(struct pci_dev *pdev)
```

**禁用逻辑：**
1. 读取ROM BAR值
2. 清除使能位
3. 写入新值

### 2. ROM大小检测

#### pci_get_rom_size
确定ROM镜像的实际大小：

```c
static size_t pci_get_rom_size(struct pci_dev *pdev, void __iomem *rom, size_t size)
```

**检测流程：**
1. 检查PCI ROM签名（0xAA55）
2. 读取PCI数据结构（PDS）
3. 验证"PCIR"签名
4. 返回镜像长度

**PCI ROM格式：**
```
偏移   大小    内容
0x00    2       0xAA55（签名）
0x02    2       0x5043（'PC' + 'IR'）
0x18    2       镜度指示器
0x1A    2       数据结构指针
```

**镜像指示器：**
- 0xFFFD: 最后一个镜像
- 0xFFFC: 压缩镜像（需要解压）

### 3. ROM映射

#### pci_map_rom
映射PCI ROM到内核空间：

```c
void __iomem *pci_map_rom(struct pci_dev *pdev, size_t length)
```

**映射过程：**
1. 获取ROM资源
2. 使用 `iomap()` 映射
3. 使能ROM解码
4. 验证ROM签名
5. 计算实际ROM大小
6. 返回映射地址

**注意事项：**
- 映射失败时返回NULL
- 需要调用者处理释放

#### pci_unmap_rom
解除ROM映射：

```c
void pci_unmap_rom(struct pci_dev *pdev)
```

**解除逻辑：**
1. 禁用ROM解码
2. 释放映射的内存

### 4. ROM读取

#### pci_read_rom
从PCI ROM读取数据：

```c
ssize_t pci_read_rom(struct pci_dev *pdev, void *buf, size_t len, size_t offset)
```

**读取方式：**
1. 先尝试从ROM资源读取
2. 如果失败，从shadow copy读取
3. 支持部分读取（任意offset和长度）

### 5. ROM复制

#### pci_copy_rom
复制ROM到缓冲区：

```c
int pci_copy_rom(struct pci_dev *pdev, bool write)
```

**复制流程：**
1. 使能ROM
2. 映射整个ROM
3. 检查ROM大小
4. 验证ROM签名
5. 复制到分配的缓冲区
6. 标记为shadow copy
7. 解除映射

## ROM资源标志

### IORESOURCE_ROM_ENABLE
```
#define IORESOURCE_ROM_ENABLE  (1UL << 2)
```
- 表示ROM已使能
- 在ROM BAR的第2位

### IORESOURCE_ROM_SHADOW
```
#define IORESOURCE_ROM_SHADOW    (1UL << 3)
```
- 表示ROM为shadow copy
- ROM内容在RAM中而非物理ROM

## 重要概念

### 1. ROM BAR格式
- **低31位**: ROM地址
- **第2位**: ROM使能
- **第3位**: shadow copy标志

### 2. PCI ROM镜像格式
- 符合x86执行码格式
- 包含镜像信息表
- 支持多个镜像链

### 3. 使能与映射
- 映射前必须使能ROM解码
- 解除映射前应禁用ROM解码
- 防止ROM与MMIO冲突

### 4. Buggy ROM处理
- 某些设备在禁用时ROM BAR读为0
- 使能时需要保留地址信息
- 通过读取-修改-写处理

## 关键要点

1. **ROM签名验证**: 检查0xAA55和"PCIR"
2. **使能位控制**: 通过ROM BAR第2位使能/禁用
3. **Shadow Copy优化**: RAM中的ROM访问更快
4. **多镜像支持**: ROM可包含多个代码镜像
5. **资源管理**: 使用标准PCI资源机制
