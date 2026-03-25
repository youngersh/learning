# PCI中断处理分析 (irq.c)

## 文件概述
`irq.c` 实现PCI中断线的分配、管理和INTx路由功能。

## 主要功能函数

### 1. 中断请求与释放

#### pci_request_irq
为PCI设备请求中断线：

```c
int pci_request_irq(struct pci_dev *dev, unsigned int nr, irq_handler_t handler,
                 irq_handler_t thread_fn, void *dev_id, const char *fmt, ...)
```

**参数说明：**
- `dev`: PCI设备
- `nr`: 设备相对中断向量索引（0-based）
- `handler`: 主中断处理函数
- `thread_fn`: 中断线程函数（可以为NULL）
- `dev_id`: 传递给处理函数的cookie
- `fmt`: 格式化字符串用于命名中断

**实现逻辑：**
1. 设置IRQ标志（`IRQF_SHARED`，无handler时设`IRQF_ONESHOT`）
2. 格式化设备名称
3. 调用 `request_threaded_irq()` 请求中断
4. 成功后保留设备名称，失败则释放

#### pci_free_irq
释放之前请求的中断：

```c
void pci_free_irq(struct pci_dev *dev, unsigned int nr, void *dev_id)
```

**释放流程：**
1. 调用 `free_irq()` 释放中断
2. 释放设备名称字符串

**注意事项：**
- 中断必须先禁用
- 不能在中断上下文中调用
- 等待所有执行中的中断完成

### 2. INTx路由

#### pci_swizzle_interrupt_pin
对桥接后的设备执行INTx交换：

```c
u8 pci_swizzle_interrupt_pin(const struct pci_dev *dev, u8 pin)
```

**交换公式：**
```c
(((pin - 1) + slot) % 4) + 1
```

**工作原理：**
- PCI规范要求INTx引脚在经过每个桥接时进行"swizzle"
- Slot号从 `devfn` 提取：`PCI_SLOT(dev->devfn)`
- ARI（Alternative Routing ID）使能时slot为0

**示例：**
```
设备INTA → 桥槽0 → INTA
设备INTA → 桥槽1 → INTB
设备INTA → 桥槽2 → INTC
设备INTA → 桥槽3 → INTD
```

#### pci_get_interrupt_pin
获取设备的中断引脚到根桥接：

```c
int pci_get_interrupt_pin(struct pci_dev *dev, struct pci_dev **bridge)
```

**处理流程：**
1. 从设备读取INT_PIN寄存器
2. 向上遍历桥接树
3. 每层对pin进行swizzle
4. 返回最终的pin和根桥接

**返回值：**
- `> 0`: 有效的INTx引脚（1-4对应INTA-D）
- `-1`: 设备不使用中断

#### pci_common_swizzle
对设备进行完整的INTx交换到根桥接：

```c
u8 pci_common_swizzle(struct pci_dev *dev, u8 *pinp)
```

**功能：**
1. 从设备开始向上遍历所有桥接
2. 每层执行INTx交换
3. 返回最终slot号和更新pin值

### 3. 中断分配

#### pci_assign_irq
分配设备的中断向量：

```c
void pci_assign_irq(struct pci)
```

**分配策略：**
1. 获取host bridge
2. 检查是否有 `map_irq` 回调
3. 如果有，调用平台特定的IRQ映射
4. 如果没有，调用 `pci_assign_legacy_irq()`

#### pci_assign_legacy_irq
传统PCI IRQ分配：

```c
static void pci_assign_legacy_irq(struct pci_dev *dev)
```

**传统IRQ范围：** 3-7, 9-12, 14-15

**分配逻辑：**
1. 读取INTx寄存器确定使用的IRQ线
2. 检查IRQ是否有效（0xFF表示无）
3. 设置设备的 `irq` 成员
4. 更新IRQ使用统计

## 架构回调

### pcibios_alloc_irq/free_irq
```c
int __weak pcibios_alloc_irq(struct pci_dev *dev)
void __weak pcibios_free_irq(struct pci_dev *dev, int irq)
```

**用途：**
- 允许平台提供自定义IRQ分配
- ARM平台常使用此机制

## INTx路由示例

### 单层桥接
```
设备        桥接
INTA (1) → 槽0 → INTA (1)
```

### 多层桥接
```
设备        桥接1        桥接2
INTA (1) → 槽0 → INTB (2) → 槽2 → INTD (4)
```

**计算过程：**
1. `(((1-1) + 0) % 4) + 1 = 1` (INTA→INTA)
2. `(((1-1) + 2) % 4) + 1 = 2` (INTA→INTB)
3. `(((2-1) + 2) % 4) + 1 = 4` (INTB→INTD)

## 关键概念

### 1. INTx引脚编号
- INTA = 1
- INTB = 2
- INTC = 3
- INTD = 4

### 2. INTx交换规则
- 每个桥接层根据slot号重新映射INTx
- 公式：`new_pin = ((old_pin - 1 + slot) % 4) + 1`
- 确保不同槽的设备使用不同的IRQ线

### 3. 中断类型
- **传统中断**: INTA-D，线共享
- **MSI**: Message Signaled Interrupts，独立中断线
- **MSI-X**: 扩展MSI，每个向量独立

### 4. 桥接Slot
- PCI slot号从 `devfn` 提取：`(devfn >> 3) & 0x1F`
- ARI使能时slot固定为0

## 关键要点

1. **INTx交换是PCI规范要求**：防止所有设备使用相同IRQ
2. **传统IRQ范围受限**：仅3-7, 9-12, 14-15可用
3. **MSI优于传统中断**：更好的性能和可扩展性
4. **threaded中断支持**：handler在线程中执行
5. **平台自定义**：通过pcibios回调支持不同架构
