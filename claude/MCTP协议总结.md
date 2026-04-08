# MCTP协议总结

## 概述
MCTP (Management Controller Transport Protocol) 是 DMTF (DSP0237 Management Component Transport Protocol) 规范的Linux内核实现，用于通过 I2C 总线管理 I2C (Inter-IC) 设备。

## 关键规范
- **DMTF (DSP0237 Management Component Transport Protocol)**: MCTP基于此规范实现
- **NCB-PCI-Express_Base 7.0**: PCIe Base规范用于底层传输

## 协议关系
```
I2C设备
    ↑
    │ MCTP协议
    │
I2C总线 ──┤ 管理控制器 ──┤ PCIe EP设备 (如适用)
```

## Linux实现位置

### 核心文件
- **drivers/net/mctp/mctp-i2c.c**: MCTP协议主实现

### 相关依赖
- **include/linux/i2c.h**: I2C总线接口
- **include/linux/i2c-mux.h**: I2C多路复用
- **include/net/mctp.h**: MCTP相关定义

## 主要数据结构

### mctp_i2c_dev (I2C设备)
```c
struct mctp_i2c_dev {
    struct net_device *ndev;          // net_device
    struct i2c_adapter *adapter;        // I2C适配器
    struct mctp_i2c_client *client;      // MCTP客户端
    struct list_head list;               // mctp_i2c_client列表
    size_t rx_pos;                     // 接收位置
    u8 rx_buffer[MCTP_I2C_BUFSZ];     // 接收缓冲区
    struct completion rx_done;            // 接收完成
    struct task_struct *tx_thread;          // 发送线程
    wait_queue_head_t tx_wq;              // 发送工作队列
    struct sk_buff_head tx_queue;          // 发送队列
    u8 tx_scratch[MCTP_I2C_BUFSZ];     // 发送缓冲区
    spinlock_t lock;                     // 自旋锁
    int i2c_lock_count;                 // I2C锁计数
    int release_count;                   // 释放计数
    bool allow_rx;                      // 允许接收
};
```

### mctp_i2c_client (I2C客户端)
```c
struct mctp_i2c_client {
    struct i2c_client *client;          // I2C客户端
    u8 lladdr;                        // 本地链路地址
    struct mctp_i2c_dev *sel;         // 选中的设备
    struct list_head devs;              // 设备列表
    spinlock_t sel_lock;                 // 保护sel和devs的锁
    struct list_head list;               // driver_clients列表
};
```

## 主要功能

### 1. 设备管理
- **mctp_i2c_new_client()**: 创建新的MCTP客户端
  - 检查7位I2C地址要求
  - 查找根适配器
  - 注册I2C slave回调

- **mctp_i2c_free_client()**: 释放MCTP客户端
  - 取消I2C slave注册
  - 清理设备列表
  - 释放资源

### 2. 消息接收
- **mctp_i2c_recv()**: 接收并处理MCTP消息
  - 消息完整性检查
  - CRC校验
  - 创建sk_buff
  - 调用netif_rx接收

**消息接收流程**：
```
接收MCTP消息
    ↓
1. 检查长度有效性
    byte_count + 1 < MCTP_I2C_MINLEN
2. 验证PE CEC
    检查byte_count与recvlen匹配
    计算并比较PEC
3. 分配sk_buff
    netdev_alloc_skb()
4. 设置协议头
    skb->protocol = htons(ETH_P_MCTP)
5. 提取数据
    skb_put_data(skb, rx_buffer, recvlen)
6. 提交网络层
    netif_rx(skb)
```

### 3. 消息发送
- **mctp_i2c_xmit()**: 通过I2C总线发送MCTP消息
  - 获取flow状态
  - 验证flow key
  - 构造MCTP消息头
  - 调用I2C slave发送

**flow状态机**：
```
MCTP_TX_FLOW_INVALID: 初始/无效状态
MCTP_TX_FLOW_NONE: 无flow
MCTP_TX_FLOW_NEW: 新flow，准备激活
MCTP_TX_FLOW_EXISTING: 流存在，可发送
MCTP_TX_FLOW_ACTIVE: 流激活
```

### 4. Flow控制
- **mctp_i2c_get_tx_flow_state()**: 获取flow状态
  - 检查skb扩展
  - 获取flow key
  - 状态转换

**状态转换**：
```
NEW → ACTIVE: flow首次创建
ACTIVE → EXISTING: flow已建立，可发送
EXISTING → INVALID: flow释放/超时
```

### 5. 锁机制
- **mctp_i2c_lock_nest()**: 获取I2C总线段锁
  - 递增i2c_lock_count
  - 调用i2c_lock_bus()

- **mctp_i2c_unlock_nest()**: 释放I2C总线段锁
  - 递减i2c_lock_count
  - 调用i2c_unlock_bus()

- **mctp_i2c_unlock_reset()**: 重置锁状态

### 6. 设备选择
- **__mctp_i2c_device_select()**: 切换I2C设备
  - 持有sel_lock保护
  - 保存当前选中的设备
  - 选择新设备

### 7. 事件处理
- **mctp_i2c_slave_cb()**: I2C slave回调
  - 处理WRITE_RECEIVED: 数据写入
  - 处理WRITE_REQUESTED: 写请求
  - 处理STOP: 停止

## 与PCIe的潜在关系

### 可能的连接场景
```
MCTP协议 (I2C)
    ↓
I2C总线
    ↓
I2C控制器
    ↓
PCIe Endpoint Device (作为I2C slave)
```

### MCTP在PCIe场景下的可能用途
1. **配置管理**：MCTP可用于配置PCIe EP设备的特定参数
2. **固件更新**：通过I2C总线向PCIe EP设备传输固件
3. **调试/诊断**：MCTP消息可用于诊断PCIe EP设备状态
4. **热插拔通知**：MCTP可通知PCIe EP设备状态变化

### 但需要注意
- **MCTP不是PCIe专用协议**：它是通用I2C总线管理协议
- **当前实现中没有看到PCIe特定的EP管理代码**
- **如果需要PCIe EP管理**，应该使用PCIe子系统专门的接口

## 配置选项

### 内核配置
```c
#define MCTP_I2C_MAXBLOCK 255        // 最大MCTP块大小
#define MCTP_I2C_MAXMTU (MCTP_I2C_MAXBLOCK - 1)  // 最大MTU
#define MCTP_I2C_BUFSZ (3 + MCTP_I2C_MAXBLOCK + 1)  // 消息缓冲区大小
#define MCTP_I2C_MINLEN 8          // 最小消息长度
#define MCTP_I2C_TX_WORK_LEN 100   // TX工作长度
#define MCTP_I2C_TX_QUEUE_LEN 1100   // TX队列长度
```

### 调试和统计
```c
struct net_device_stats {
    rx_packets;      // 接收包计数
    rx_bytes;         // 接收字节数
    rx_dropped;       // 丢弃包计数
    rx_length_errors;  // 长度错误计数
    rx_crc_errors;    // CRC错误计数
    rx_over_dropped; // 超长错误计数
};
```

## 协议消息格式

### 消息头 (mctp_i2c_hdr)
```c
struct mctp_i2c_hdr {
    u8 dest_slave;    // 目标slave地址
    u8 command;       // 命令
    u8 byte_count;     // 字节计数(不含PEC)
    u8 source_slave;  // 源slave地址
};
```

### 消息格式
```
[dest_slave][command][byte_count][data] [PEC]
```

### 常见命令
- READ/WRITE: 数据传输
- 响应消息：包含数据 + PEC

## 关键要点

1. **MCTP是I2C总线协议，不是PCIe专用**
   - 当前实现主要管理I2C设备
   - 没有看到PCIe EP的专用管理代码

2. **如果需要PCIe EP管理，应该使用**：
   - `drivers/pci/endpoint/` - PCIe Endpoint框架
   - `drivers/pci/endpoint/pci-epc-core.c` - Endpoint Controller核心
   - `drivers/pci/endpoint/pci-epc-mem.c` - 内存管理
   - `drivers/pci/endpoint/pci-epc-ops.c` - 操作接口

3. **MCTP可作为传输层**：
   - 如果需要通过MCTP协议管理PCIe EP设备
   - 需要在MCTP客户端中实现PCIe EP特定逻辑

4. **消息完整性保护**：
   - CRC校验
   - 长度验证
   - PEC (Packet Error Check)

5. **流控制机制**：
   - 防止并发发送
   - 管理flow生命周期
