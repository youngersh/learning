# Advanced Error Reporting (AER) Capability 详细分析

## 概述

**Extended Capability ID**: PCI_EXT_CAP_ID_ERR (0x01)  
**规范参考**: PCIe Advanced Error Reporting Specification  
**功能**: 报告和恢复 PCIe 错误（可纠正和不可纠正）

## PCIe SPEC 定义

### Extended Capability 结构
```
Offset  Size  Description
0x00    4    Extended Capability Header
              - Bits [15:0]: Capability ID (0x01)
              - Bits [19:16]: Capability Version
              - Bits [31:20]: Next Capability Offset
0x04    4    Uncorrectable Error Status
0x08    4    Uncorrectable Error Mask
0x0C    4    Uncorrectable Error Severity
0x10    4    Correctable Error Status
0x14    4    Correctable Error Mask
0x18    4    Advanced Error Capabilities and Control
0x1C    16   Header Log Register
0x30    4    Root Error Command (Root Ports only)
0x34    4    Root Error Status (Root Ports only)
0x38    4    Error Source Identification (Root Ports only)
```

### Uncorrectable Error Status Bits
| Bit | 错误 | 描述 |
|-----|------|------|
| 4 | Data Link Protocol | 数据链路协议错误 |
| 5 | Surprise Down | 意外断开 |
| 12 | Poisoned TLP | 中毒 TLP |
| 13 | Flow Control Protocol | 流控协议错误 |
| 14 | Completion Timeout | 完成超时 |
| 15 | Completer Abort | 完成者中止 |
| 16 | Unexpected Completion | 意外完成 |
| 17 | Receiver Overflow | 接收器溢出 |
| 18 | Malformed TLP | 格式错误 TLP |
| 19 | ECRC Error | ECRC 错误 |
| 20 | Unsupported Request | 不支持的请求 |
| 21 | ACS Violation | ACS 违规 |
| 22 | Internal Error | 内部错误 |
| 23 | MC Blocked TLP | MC 阻塞 TLP |
| 24 | Atomic Egress Blocked | 原子出口阻塞 |
| 25 | TLP Prefix Blocked | TLP 前缀阻塞 |
| 26 | Poisoned TLP Egress Blocked | 中毒 TLP 出口阻塞 |
| 27 | DMWr Request Egress Blocked | DMWr 请求出口阻塞 |
| 28 | IDE Check Failed | IDE 检查失败 |
| 29 | Misrouted IDE TLP | 错误路由 IDE TLP |
| 30 | PCRC Check Failed | PCRC 检查失败 |
| 31 | TLP Translation Egress Blocked | TLP 转换出口阻塞 |

### Correctable Error Status Bits
| Bit | 错误 | 描述 |
|-----|------|------|
| 0 | Receiver Error | 接收器错误 |
| 6 | Bad TLP | 错误 TLP |
| 7 | Bad DLLP | 错误 DLLP |
| 8 | Replay Timer Rollover | 重放计时器溢出 |
| 12 | Replay Timer Timeout | 重放计时器超时 |
| 13 | Advisory Non-Fatal | 建议非致命错误 |
| 14 | Corrected Internal Error | 已纠正内部错误 |
| 15 | Header Log Overflow | 头部日志溢出 |

## Linux 内核实现

### 初始化流程

**文件**: `drivers/pci/pcie/aer.c`  
**函数**: `pci_aer_init()` (行 388-422)

```c
void pci_aer_init(struct pci_dev *dev)
{
    int n;

    // 1. 查找 AER capability
    dev->aer_cap = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ERR);
    if (!dev->aer_cap)
        return;

    // 2. 分配 AER 信息结构
    dev->aer_info = kzalloc(sizeof(*dev->aer_info), GFP_KERNEL);
    if (!dev->aer_info) {
        dev->aer_cap = 0;
        return;
    }

    // 3. 初始化速率限制
    ratelimit_state_init(&dev->aer_info->correctable_ratelimit,
                     DEFAULT_RATELIMIT_INTERVAL, DEFAULT_RATELIMIT_BURST);
    ratelimit_state_init(&dev->aer_info->nonfatal_ratelimit,
                     DEFAULT_RATELIMIT_INTERVAL, DEFAULT_RATELIMIT_BURST);

    // 4. 分配保存缓冲区
    n = pcie_cap_has_rtctl(dev) ? 5 : 4;
    pci_add_ext_cap_save_buffer(dev, PCI_EXT_CAP_ID_ERR, sizeof(u32) * n);

    // 5. 清除 AER 状态
    pci_aer_clear_status(dev);

    // 6. 启用 AER
    if (pci_aer_available_available())
        pci_enable_pcie_error_reporting(dev);

    // 7. 配置 ECRC
    pcie_set_ecrc_checking(dev);
}
```

### 启用 AER

```c
void pci_enable_pcie_error_reporting(struct pci_dev *dev)
{
    u32 aer, aer_mask;

    // 1. 读取 AER Capabilities
    pci_read_config_dword(dev, dev->aer_cap + PCI_ERR_CAP, &aer);

    // 2. 配置错误掩码
    aer_mask = 0;

    // 可纠正错误
    if (aer & PCI_ERR_CAP_ECRC_GENC)
        aer_mask |= PCI_ERR_COR_ECRC;

    // 不可纠正错误
    if (aer & PCI_ERR_CAP_ECRC_CHKC)
        aer_mask |= PCI_ERR_UNC_ECRC;

    // 3. 写入错误掩码
    pci_write_config_dword(dev, dev->aer_cap + PCI_ERR_COR_MASK,
                       aer_mask);
    pci_write_config_dword(dev, dev->aer_cap + PCI_ERR_UNCOR_MASK,
                       aer_mask);

    // 4. 配置严重性
    pci_write_config_dword(dev, dev->aer_cap + PCI_ERR_UNCOR_SEVER,
                       0);  // 全部设为非致命

    // 5. Root Port 特殊处理
    if (pcie_cap_has_rtctl(dev)) {
        u32 root_cmd;

        pci_read_config_dword(dev, dev->aer_cap + PCI_ERR_ROOT_COMMAND,
    &root_cmd);

        // 启用错误报告
        root_cmd |= (PCI_ERR_ROOT_CMD_COR_EN |
                   PCI_ERR_ROOT_CMD_NONFATAL_EN |
                   PCI_ERR_ROOT_CMD_FATAL_EN);

        pci_write_config_dword(dev, dev->aer_cap + PCI_ERR_ROOT_COMMAND,
                           root_cmd);
    }
}
```

### 错误处理

#### 1. aer_irq()
```c
irqreturn_t aer_irq(int irq, void *context)
{
    struct pcie_device *pdev = context;
    struct pci_dev *dev = pdev->port;
    int aer = dev->aer_cap;
    u32 status, events;

    // 1. 读取 Root Error Status
    pci_read_config_dword(dev, aer + PCI_ERR_ROOT_STATUS, &status);

    // 2. 检查是否有错误
    if (status == 0)
        return IRQ_NONE;

    // 3. 获取错误事件
    events = status & (PCI_ERR_ROOT_COR_RCV |
                   PCI_ERR_ROOT_UNCOR_RCV);

    // 4. 清除状态
    pci_write_config_dword(dev, aer + PCI_ERR_ROOT_STATUS, status);

    // 5. 处理错误
    if (events & PCI_ERR_ROOT_COR_RCV)
        aer_process_correctable_errors(dev);

    if (events & PCI_ERR_ROOT_UNCOR_RCV)
        aer_process_uncorrectable_errors(dev);

    return IRQ_HANDLED;
}
```

#### 2. aer_process_uncorrectable_errors()
```c
void aer_process_uncorrectable_errors(struct pci_dev *dev)
{
    int aer = dev->aer_cap;
    u32 status, mask, sev;
    struct aer_err_info info;

    // 1. 读取错误状态
    pci_read_config_dword(dev, aer + PCI_ERR_UNCOR_STATUS, &status);
    pci_read_config_dword(dev, aer + PCI_ERR_UNCOR_MASK, &mask);
    pci_read_config_dword(dev, aer + PCI_ERR_UNCOR_SEVER, &sev);

    // 2. 过滤已屏蔽的错误
    status &= ~mask;

    // 3. 检查致命错误
    if (status & ~sev) {
        // 致命错误
        pci_err(dev, "Fatal AER error: %#x\n", status);
        pcie_do_recovery(dev, pci_channel_io_frozen, NULL);
        return;
    }

    // 4. 非致命错误
    if (status & sev) {
        pci_err(dev, "Non-fatal AER error: %#x\n", status);
        aer_print_error(dev, &info);
    }
}
```

#### 3. aer_process_correctable_errors()
```c
void aer_process_correctable_errors(struct pci_dev *dev)
{
    int aer = dev->aer_cap;
    u32 status, mask;
    struct aer_err_info info;

    // 1. 读取错误状态
    pci_read_config_dword(dev, aer + PCI_ERR_COR_STATUS, &status);
    pci_read_config_dword(dev, aer + PCI_ERR_COR_MASK, &mask);

    // 2. 过滤已屏蔽的错误
    status &= ~mask;

    // 3. 检查是否有错误
    if (status == 0)
        return;

    // 4. 速率限制
    if (!__ratelimit(&dev->aer_info->correctable_ratelimit))
        return;

    // 5. 打印错误
    pci_info(dev, "Correctable AER error: %#x\n", status);
    aer_print_error(dev, &info);

    // 6. 清除状态
    pci_write_config_dword(dev, aer + PCI_ERR_COR_STATUS, status);
}
```

### 错误恢复

```c
pci_ers_result_t pcie_do_recovery(struct pci_dev *dev,
        pci_channel_state_t state,
        pci_ers_result_t (*reset_subordinates)(struct pci_dev *pdev))
{
    pci_ers_result_t status = PCI_ERS_RESULT_DISCONNECT;

    // 1. 通知驱动
    status = pci_report_error(dev, state);

    // 2. 如果驱动支持恢复
    if (status == PCI_ERS_RESULT_CAN_RECOVER) {
        // 3. 执行恢复
        status = pci_broadcast_error_message(dev,
                    PCI_CHANNEL_IO_FROZEN);

        if (status == PCI_ERS_RESULT_NEED_RESET) {
            // 4. 链路重置
            status = pci_reset_link(dev, state);

            if (status == PCI_ERS_RESULT_RECOVERED) {
                // 5. 通知驱动恢复完成
                pci_broadcast_error_message(dev,
                    PCI_CHANNEL_IO_NORMAL);
            }
        }
    }

    return status;
}
```

## Root Port 处理

Root Port 作为错误报告的汇聚点，需要特殊处理：

```c
// 检查是否是 Root Port
if (pci_pcie_type(dev) == PCI_EXP_TYPE_ROOT_PORT) {
    // Root Port 有额外的 Root Error 寄存器
    u32 root_cmd, root_status;

    pci_read_config_dword(dev, aer + PCI_ERR_ROOT_COMMAND, &root_cmd);
    pci_read_config_dword(dev, aer + PCI_ERR_ROOT_STATUS, &root_status);

    // 启用错误报告
    root_cmd |= (PCI_ERR_ROOT_CMD_COR_EN |
                   PCI_ERR_ROOT_CMD_NONFATAL_EN |
                   PCI_ERR_ROOT_CMD_FATAL_EN);

    pci_write_config_dword(dev, aer + PCI_ERR_ROOT_COMMAND, root_cmd);
}
```

## 设备驱动使用

### 1. 注册错误处理回调
```c
static pci_ers_result_t my_error_detected(struct pci_dev *pdev,
                                      pci_channel_state_t state)
{
    switch (state) {
    case pci_channel_io_normal:
        // 链路正常
        break;
    case pci_channel_io_frozen:
        // 链路冻结，需要恢复
        return PCI_ERS_RESULT_NEED_RESET;
    case pci_channel_io_perm_failure:
        // 永久失败
        return PCI_ERS_RESULT_DISCONNECT;
    }

    return PCI_ERS_RESULT_NONE;
}

// 注册回调
pci_enable_pcie_error_reporting(pdev);
```

### 2. 实现恢复操作
```c
static pci_ers_result_t my_slot_reset(struct pci_dev *pdev)
{
    // 1. 保存设备状态
    pci_save_state(pdev);

    // 2. 重置链路
    pci_reset_function(pdev);

    // 3. 恢复设备状态
    pci_restore_state(pdev);

    // 4. 重新初始化设备
    my_device_reinit(pdev);

    return PCI_ERS_RESULT_RECOVERED;
}

// 注册恢复回调
pci_set_drvdata(pdev, my_drvdata);
```

### 3. 处理 AER 中断
```c
// 如果设备需要处理 AER 中断
static irqreturn_t my_aer_handler(int irq, void *dev_id)
{
    struct pci_dev *pdev = dev_id;

    // 读取 AER 状态
    u32 status;
    pci_read_config_dword(pdev, pdev->aer_cap + PCI_ERR_UNCOR_STATUS,
                       &status);

    // 处理错误
    if (status) {
        // 清除状态
        pci_write_config_dword(pdev, pdev->aer_cap + PCI_ERR_UNCOR_STATUS,
                           status);
    }

    return IRQ_HANDLED;
}

// 注册中断
request_irq(pdev->irq, my_aer_handler, IRQF_SHARED, "my_aer", pdev);
```

## 调用流程

```
pci_scan_single_device()
  └─> pci_scan_device()
       └─> pci_setup_device()
  └─> pci_device_add()
       └─> pci_init_capabilities()
            └─> pci_aer_init()
                 ├─> pci_find_ext_capability(PCI_EXT_CAP_ID_ERR)
                 ├─> kzalloc()  // 分配 aer_info
                 ├─> ratelimit_state_init()  // 初始化速率限制
                 ├─> pci_add_ext_cap_save_buffer()  // 分配保存缓冲区
                 ├─> pci_aer_clear_status()  // 清除状态
                 ├─> pci_enable_pcie_error_reporting()  // 启用 AER
                 └─> pcie_set_ecrc_checking()  // 配置 ECRC

错误发生:
  └─> aer_irq()
       ├─> pci_read_config_dword()  // 读取 Root Error Status
       ├─> aer_process_correctable_errors()  // 处理可纠正错误
       └─> aer_process_uncorrectable_errors()  // 处理不可纠正错误
            └─> pcie_do_recovery()  // 执行恢复
                 ├─> pci_report_error()  // 通知驱动
                 ├─> pci_broadcast_error_message()  // 广播错误
                 └─> pci_reset_link()  // 重置链路
```

## 配置空间访问

### 查找 AER Capability
```c
int aer_pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ERR);
```

### 读取错误状态
```c
u32 uncor_status, cor_status;

pci_read_config_dword(dev, aer_pos + PCI_ERR_UNCOR_STATUS, &uncor_status);
pci_read_config_dword(dev, aer_pos + PCI_ERR_COR_STATUS, &cor_status);
```

### 配置错误掩码
```c
u32 uncor_mask = 0;
u32 cor_mask = 0;

// 屏蔽某些错误
uncor_mask |= PCI_ERR_UNC_DLP;
uncor_mask |= PCI_ERR_UNC_POISON_TLP;

cor_mask |= PCI_ERR_COR_BAD_TLP;
cor_mask |= PCI_ERR_COR_BAD_DLLP;

pci_write_config_dword(dev, aer_pos + PCI_ERR_UNCOR_MASK, uncor_mask);
pci_write_config_dword(dev, aer_pos + PCI_ERR_COR_MASK, cor_mask);
```

### 配置严重性
```c
u32 severity = 0;

// 设置某些错误为致命
severity |= PCI_ERR_UNC_COMP_ABORT;
severity |= PCI_ERR_UNC_UNX_COMP;

pci_write_config_dword(dev, aer_pos + PCI_ERR_UNCOR_SEVER, severity);
```

### Root Error 控制
```c
u32 root_cmd;

pci_read_config_dword(dev, aer_pos + PCI_ERR_ROOT_COMMAND, &root_cmd);

// 启用错误报告
root_cmd |= PCI_ERR_ROOT_CMD_COR_EN;      // 可纠正错误
root_cmd |= PCI_ERR_ROOT_CMD_NONFATAL_EN; // 非致命错误
root_cmd |= PCI_ERR_ROOT_CMD_FATAL_EN;     // 致命错误

pci_write_config_dword(dev, aer_pos + PCI_ERR_ROOT_COMMAND, root_cmd);
```

## 错误恢复流程

```
错误发生
    ↓
AER 中断触发
    ↓
读取错误状态
    ↓
判断错误类型
    ├─ 可纠正错误 → 清除状态 → 继续
    └─ 不可纠正错误
           ↓
       判断严重性
       ├─ 非致命错误 → 通知驱动 → 尝试恢复
       └─ 致命错误 → 通知驱动 → 重置链路
              ↓
         驱动恢复
              ↓
         恢复成功 → 继续
         恢复失败 → 断开连接
```

## 性能考虑

1. **速率限制**: 防止错误日志洪水
2. **错误掩码**: 减少中断频率
3. **恢复时间**: 恢复操作可能耗时较长
4. **Root Port 汇聚**: 所有错误最终报告到 Root Port

## 调试信息

```c
// 错误打印
pci_err(dev, "Fatal AER error: %#x\n", status);
pci_info(dev, "Correctable AER error: %#x\n", status);

// 错误详情
aer_print_error(dev, &info);
```

## 错误处理

1. **AER 不支持**:
   ```c
   if (!dev->aer_cap)
       return;
   ```

2. **内存分配失败**:
   ```c
   dev->aer_info = kzalloc(sizeof(*dev->aer_info), GFP_KERNEL);
   if (!dev->aer_info) {
       dev->aer_cap = 0;
       return;
   }
   ```

3. **恢复失败**:
   ```c
   status = pcie_do_recovery(dev, state, reset_fn);
   if (status != PCI_ERS_RESULT_RECOVERED) {
       pci_err(dev, "AER recovery failed\n");
   }
   ```

## 相关配置选项

- `CONFIG_PCIEAER`: 启用 AER 支持
- `CONFIG_PCIEAER_INJECT`: AER 错误注入（测试用）
- `CONFIG_PCIE_ECRC`: ECRC 支持

## 参考资料

- PCIe Advanced Error Reporting Specification
- PCIe Base Specification Revision 5.0+, Section 6.2
- Linux 内核源码: `drivers/pci/pcie/aer.c`
- 头文件: `include/linux/aer.h`, `include/uapi/linux/pci_regs.h`
