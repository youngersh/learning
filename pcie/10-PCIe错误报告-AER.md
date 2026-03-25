# PCIe错误报告-AER分析 (aer.c)

## 文件概述
`aer.c` 实现PCIe Advanced Error Reporting (AER) 功能，处理PCIe链路的错误检测和报告。

## 核心数据结构

### aer_err_source
```c
struct aer_err_source {
    u32 status;     // 根端口状态 (PCI_ERR_ROOT_STATUS)
    u32 id;         // 错误源ID (PCI_ERR_ROOT_ERR_SRC)
};
```

### aer_rpc
```c
struct aer_rpc {
    struct pci_dev *rpd;                 // 根端口设备
    DECLARE_KFIFO(aer_fifo, struct aer_err_source, AER_ERROR_SOURCES_MAX);
};
```
- 使用KFIFO缓存错误源
- 最大128个错误源

### aer_info
```c
struct aer_info {
    // 设备级错误计数
    u64 dev_cor_errs[AER_MAX_TYPEOF_COR_ERRS];
    u64 dev_fatal_errs[AER_MAX_TYPEOF_UNCOR_ERRS];
    u64 dev_nonfatal_errs[AER_MAX_TYPEOF_UNCOR_ERRS];

    // 总错误计数
    u64 dev_total_cor_errs;
    u64 dev_total_fatal_errs;
    u64 dev_total_nonfatal_errs;

    // 根端口统计
    u64 rootport_total_cor_errs;
    u64 rootport_total_fatal_errs;
    u64 rootport_total_nonfatal_errs;

    // 速率限制
    struct ratelimit_state correctable_ratelimit;
    struct ratelimit_state nonfatal_ratelimit;
};
```

## 主要功能函数

### 1. AER初始化与控制

#### pci_no_aer
禁用AER功能：

```c
void pci_no_aer(void)
```
- 设置全局禁用标志

#### pci_aer_available
检查AER是否可用：

```c
bool pci_aer_available(void)
```
- 需要未禁用且MSI已使能

### 2. 根端口服务

#### aer_service
AER服务绑定：

```c
static struct pcie_port_service_driver aer_service = {
    .name = "aer",
    .port_type = PCIE_ANY_PORT,
    .service = PCIE_PORT_SERVICE_AER,
    .probe = aer_probe,
    .remove = aer_remove,
};
```

#### aer_probe
探测并初始化AER：

```c
static int aer_probe(struct pcie_device *service)
```

**探测流程：**
1. 检查设备是否为根端口或事件收集器
2. 验证设备AER能力
3. 分配AER信息结构
4. 初始化速率限制
5. 使能AER错误报告
6. 注册AER中断处理

#### aer_remove
移除AER服务：

```c
static void aer_remove(struct pcie_device *service)
```

**清理内容：**
1. 禁用AER错误报告
2. 释放速率限制
3. 释放AER信息结构

### 3. 中断处理

#### aer_isr
AER中断服务例程：

```c
static irqreturn_t aer_isr(int irq, void *context)
```

**处理流程：**
1. 读取根端口错误状态寄存器
2. 收集多个错误报告（支持最多32个）
3. 调度错误处理工作队列

#### aer_irq
AER中断处理（线程）：

```c
static void aer_irq(void *context)
```

**处理步骤：**
1. 收集错误源到FIFO
2. 为每个错误源处理错误

### 4. 错误处理

#### aer_process_err_sources
处理收集的错误源：

```c
static void aer_process_err_sources(struct aer_rpc *rpc)
```

**处理逻辑：**
1. 从FIFO取出错误源
2. 根据错误类型分发：
   - 正确错误：`aer_correctable_error()`
   - 不可恢复错误：`aer_unrecoverable_error()`
   - 非致命错误：`aer_nonfatal_error()`
3. 应用速率限制

### 5. 错误报告

#### aer_correctable_error
处理正确错误：

```c
static void aer_correctable_error(struct aer_err_source *e_info,
                                      struct pci_dev *dev)
```

**错误类型：**
- **接收错误** (Receiver Error)
- **坏TLP错误** (Bad TLP)
- **坏DLLP错误** (Bad DLLP)
- **重放超时错误** (Replay Timeout)
- **重放次数超限错误** (Replay Number Rollover)
- **物理层错误** (Physical Layer)
- **数据链路层错误** (Data Link Layer)

#### aer_unrecoverable_error
处理不可恢复错误：

```c
static void aer_unrecoverable_error(struct aer_err_source *e_info,
                                         struct pci_dev *dev)
```

**错误类型：**
- **Poisoned TLP错误**
- **ECRC错误** (ECCRC)
- **流控协议错误**
- **完成超时错误**

### 6. 错误统计

#### aer_stats_setup
设置错误统计debugfs：

```c
static void aer_stats_setup(struct aer_err_info *info)
```

**统计内容：**
- 设备级正确/不可恢复/非致命错误计数
- 根端口总错误计数
- 速率限制统计

## ECRC (Error Collection and Reporting)

### ECRC策略
```c
#define ECRC_POLICY_DEFAULT    0    // BIOS设置
#define ECRC_POLICY_OFF       1    // 性能优先
#define ECRC_POLICY_ON        2    // 数据完整性优先
```

### ECRC实现
- 由Root Complex Event Collectors使用
- 支持多设备错误聚合
- 提供更好的错误隔离

## 错误源

### 正确错误 (Correctable Errors)
可由硬件自动恢复：
- CRC错误
- 符号错误
- 重放超时
- 物理层错误

### 不可恢复错误 (Unrecoverable Errors)
需要链路复位：
- Poisoned TLP
- ECRC错误
- 协议错误

### 非致命错误 (Non-Fatal Errors)
记录但继续：
- 流控错误
- 完成超时错误
- 某些物理层错误

## 关键要点

1. **根端口服务**：AER作为PCIe端口服务绑定
2. **FIFO缓冲**：使用KFIFO缓存错误源
3. **速率限制**：限制错误报告频率
4. **多错误支持**：支持最多32个错误报告
5. **错误分类**：正确、不可恢复、非致命
6. **ECRC支持**：事件收集器使用不同策略
7. **统计信息**：通过debugfs提供错误统计
