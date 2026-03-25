# PCIe电源管理-ASPM分析 (aspm.c)

## 文件概述
`aspm.c` 实现PCIe链路的Active State Power Management（ASPM），管理链路电源状态。

## LTR (Latency Tolerance Reporting)

### pci_save_ltr_state
保存LTR配置：

```c
void pci_save_ltr_state(struct pci_dev *dev)
```

**保存内容：**
- LTR的最大非阻塞嗅探延迟（Max No-Snoop Latency）
- 某些broken设备只支持dword访问

### pci_restore_ltr_state
恢复LTR配置：

```c
void pci_restore_ltr_state(struct pci_dev *dev)
```

**恢复过程：**
- 写入保存的Max No-Snoop Latency值
- 某些broken设备只支持dword访问

## L1SS (Link 1.1 Substate)

### pci_configure_aspm_l1ss
配置ASPM L1.1子状态：

```c
void pci_configure_aspm_l1ss(struct pci_dev *pdev)
```

**配置内容：**
- 查找L1.1扩展能力
- 分配保存缓冲区（2×u32）
- 支持L1.0和L1.1子状态

### pci_save_aspm_l1ss_state
保存L1.1子状态配置：

```c
void pci_save_aspm_l1ss_state(struct pci_dev *pdev)
```

**保存范围：**
- Downstream Port的L1SS CTL1和CTL2
- Parent Upstream Port的L1SS CTL1和CTL2
- Downstream Port的L0s/L1子状态配置

### pci_restore_aspm_l1ss_state
恢复L1.1子状态配置：

```c
void pci_restore_aspm_l1ss_state(struct pci_dev *pdev)
```

**恢复逻辑：**
1. 检查是否为Downstream Port且存在Parent
2. 读取保存的Child L1SS配置
3. 读取保存的Parent L1SS配置
4. 恢复Downstream Port CTL2（如果BIOS使能需先禁用）
5. 恢复Downstream Port CTL1
6. 恢复Parent CTL2
7. 恢复Parent CTL1
8. 恢复Child L1SS配置

**特殊处理：**
- Downstream Port不应直接恢复L1SS状态
- 恢复应通过Upstream Port完成

## ASPM链路状态

### ASPM状态
```c
enum pcie_link_state {
    PCIE_LINK_STATE_L0S,     // L0s: 完全使能
    PCIE_LINK_STATE_L1,      // L1: 低功耗恢复时间短
    PCIE_LINK_STATE_L0_1,     // L0s/L1: 低功耗恢复时间长
    PCIE_LINK_STATE_L1_1,     // L1/L1.1: 最低功耗恢复时间最长
    PCIE_LINK_STATE_MAX
};
```

### 链路使能位
- **PCI_EXP_LNKCTL_ASPM_L0S**: 使能L0s
- **PCI_EXP_LNKCTL_ASPM_L1**: 使能L1
- **PCI_EXP_LNKCTL_ASPM_CLKPM**: 使能时钟电源管理
- **PCI_EXP_LNKCTL_CLKREQ_EN**: 使能时钟请求

## 配置原则

### 电源vs. 性能权衡
- **L0s**: 最大性能，高功耗
- **L1**: 低功耗，性能略降
- **L0s/L1**: 更低功耗，更长恢复时间
- **L1/L1.1**: 最低功耗，最长恢复时间

### 时钟电源管理
- 设备空闲时可关闭时钟
- 通过CLKREQ信号控制
- 需要上游和下游都支持

### 常见策略
- **高性能模式**: L0s，最小延迟
- **平衡模式**: L1，适中的功耗和性能
- **省电模式**: L0s/L1或L1/L1.1，最大省电

## 关键要点

1. **LTR重要性**: 影响下游设备的延迟容忍
2. **L1SS父-子协调**: 父子端口需协同配置
3. **恢复顺序**: 先恢复Parent再恢复Child
4. **时钟同步**: 时钟请求需要上游配合
5. **BIOS配置**: 某些配置由BIOS设置，内核需协调
