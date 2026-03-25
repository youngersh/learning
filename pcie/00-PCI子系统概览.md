# Linux PCI子系统概览

## 子系统架构

Linux PCI子系统是内核中PCI/PCIe设备管理的核心模块，负责：
- 设备探测和枚举
- 配置空间访问
- 资源分配和管理
- 驱动绑定和生命周期管理
- 中断处理
- 电源管理
- 错误检测和恢复

## 目录结构

```
drivers/pci/
├── access.c          # 配置空间访问
├── bus.c             # 总线资源管理
├── probe.c           # 设备探测和枚举
├── pci-driver.c       # 驱动管理
├── setup-res.c       # 设备资源（BAR）管理
├── setup-bus.c       # 总线资源分配
├── irq.c             # 中断处理
├── remove.c          # 设备移除
├── rom.c             # Option ROM管理
├── pcie/             # PCIe扩展功能
│   ├── aer.c        # 高级错误报告
│   ├── aspm.c       # 链路电源管理
│   ├── pme.c        # 电源管理事件
│   ├── dpc.c        # 下游端口包容
│   ├── ptm.c        # 精确时间测量
│   └── portdrv.c     # 端口服务驱动
├── msi/              # MSI中断支持
├── controller/        # 平台特定主机控制器
├── endpoint/          # PCIe端点模式支持
├── hotplug/           # 热插拔支持
└── switch/            # PCIe交换机支持
```

## 核心数据结构

### struct pci_dev
表示一个PCI设备：
```
vendor/device              # 厂商/设备ID
subsystem_vendor/device   # 子系统厂商/设备ID
class                    # 设备类别
devfn                    # 设备/功能号 (bus:dev.func)
bus                      # 所属PCI总线
resource[]               # BAR资源 (6标准+ROM)
irq                      # 传统INTx中断号
is_physfn/is_virtfn    # SR-IOV PF/VF标志
```

### struct pci_bus
表示一个PCI总线：
```
number                   # 总线号
parent                   # 父桥接设备
self                     # 桥接设备
children                 # 设备列表
ops                      # 架构特定操作
resource[]               # 桥接资源窗口
```

### struct pci
表示一个PCI驱动：
```
name                     # 驱动名称
id_table                # 支持的设备ID表
probe/remove             # 探测和移除回调
suspend/resume           # 电源管理回调
```

## 主要工作流程

### 1. 系统初始化
```
pci_init()
  ├── 初始化配置空间访问
  ├── 注册总线类型
  └── 扫描根总线
```

### 2. 设备枚举
```
pci_scan_slot()
  ├── 读取设备ID
  ├── 分配pci_dev结构
  ├── 设置设备信息
  ├── 探测BAR
  ├── 检测能力
  └── 扫描子总线（桥接）
```

### 3. 资源分配
```
pci_bus_assign_resources()
  ├── 第1遍历：计算所需资源
  ├── 第2遍历：分配资源到总线窗口
  └── 第3遍历：更新BAR寄存器
```

### 4. 驱动绑定
```
pci_device_probe()
  ├── 匹配设备ID
  ├── 分配IRQ
  ├── 调用驱动probe()
  └── 使能设备
```

### 5. 电源管理
```
系统suspend
  ├── 保存设备配置
  ├── 进入低功耗状态
  └── 禁用设备唤醒

系统resume
  ├── 恢复电源状态
  ├── 恢复设备配置
  └── 重新使能设备
```

### 6. 错误处理
```
AER错误报告
  ├── 检测链路错误
  ├── 分类错误类型
  ├── 记录错误统计
  └── 触执行错误恢复
```

## 关键子系统

### 配置空间访问 (access.c)
- 提供字节/字/双字配置空间读写
- 支持PCIe能力寄存器访问
- 实现访问锁保护

### 总线管理 (bus.c)
- 管理总线资源窗口
- 处理设备添加和移除
- 提供总线遍历接口

### 设备探测 (probe.c)
- PCI设备发现和枚举
- BAR检测和设置
- 设备能力检测
- 桥接扫描和子总线创建

### 驱动管理 (pci-driver.c)
- 驱动注册和注销
- 设备ID匹配
- 探测和移除回调
- 动态ID支持

### 资源分配 (setup-res.c, setup-bus.c)
- BAR资源声明和分配
- 桥接资源窗口计算
- 三遍资源分配算法

### 中断处理 (irq.c)
- 传统INTx中断分配
- INTx路由和交换
- MSI/MSI-X支持

### 设备移除 (remove.c)
- 清理设备资源
- 释放驱动引用
- 移除sysfs和proc接口

### PCIe电源管理 (pcie/aspm.c)
- ASPM (Active State Power Management)
- L0s/L1/L0s/L1/L1.1状态
- 时钟电源管理
- LTR (Latency Tolerance Reporting)

### PCIe错误报告 (pcie/aer.c)
- 高级错误报告
- 错误分类和统计
- 错误恢复机制
- ECRC支持

## PCIe扩展功能

### AER (Advanced Error Reporting)
- PCIe链路错误检测
- 可纠正和不可纠正错误处理
- 错误注入和统计

### ASPM (Active State Power Management)
- 链路电源状态管理
- L0s/L1/L0s/L1.1子状态
- 时钟电源管理

### PME (Power Management Events)
- 唤醒事件处理
- 系统唤醒检测

### DPC (Downstream Port Containment)
- 错误包容
- 链路恢复触发

### PTM (Precision Time Measurement)
- 精确时间测量
- 时间同步支持

## 重要机制

### 1. 热插拔
- ACPI事件通知
- 设备动态添加和移除
- 资源重新分配

### 2. SR-IOV (Single Root I/O Virtualization)
- 物理函数 (PF) 和虚拟函数 (VF)
- 动态VF创建和销毁
- VF资源管理

### 3. MSI/MSI-X
- 消息信号中断
- 独立中断向量
- 支持多个中断

### 4. DMA映射
- 设备DMA地址映射
- IOMMU集成
- 64位地址支持

### 5. 电源管理
- 运行时电源管理
- 系统睡眠/唤醒
- 设备电源状态转换

## 调试和诊断

### sysfs接口
```
/sys/bus/pci/devices/0000:00:00/
├── config        # 配置空间
├── resource[0-6] # BAR资源
├── enable        # 设备使能控制
├── modalias      # 模块别名
└── ...
```

### 用户空间工具
- lspci: 列出PCI设备
- setpci: 修改配置空间
- pcitest: PCIe端点测试

## 性能优化

### NUMA绑定
- 驱动probe在本地NUMA节点执行
- 优化内存访问局部性

### 资源分配优化
- 三遍分配算法
- 紧凑资源分配

### 延迟分配
- 延迟资源声明
- 支持热插拔

## 安全性

### 配置空间访问保护
- 使用锁防止并发访问
- 地址对齐检查
- 错误响应处理

### 设备隔离
- VF隔离
- DMA地址空间隔离
- 错误影响范围控制

## 参考资料

- PCI本地总线规范
- PCIe基础规范
- Linux内核文档: Documentation/PCI/
