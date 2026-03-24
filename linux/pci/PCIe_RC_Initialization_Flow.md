# PCIe RC初始化完整流程（基于ACPI）

本文档详细描述了Linux内核中PCIe Root Complex (RC)的初始化流程，重点讲解基于ACPI启动方式的完整调用链。

---

## 目录

1. [OS启动 - ACPI子系统初始化](#1-os启动---acpi子系统初始化)
2. [ACPI命名空间扫描](#2-acpi命名空间扫描)
3. [发现PCI Root Bridge](#3-发现pci-root-bridge)
4. [acpi_pci_root_add() - 创建PCI Root](#4-acpi_pci_root_add---创建pci-root)
5. [pci_acpi_scan_root() - 扫描PCI Root](#5-pci_acpi_scan_root---扫描pci-root)
6. [pci_scan_root_bus() - 扫描PCI总线](#6-pci_scan_root_bus---扫描pci总线)
7. [Platform Device方式（某些架构）](#7-platform-device方式某些架构)
8. [Platform Driver Probe](#8-platform-driver-probe)
9. [pci_host_probe() - 最终初始化](#9-pci_host_probe---最终初始化)
10. [完整调用链总结](#10-完整调用链总结)
11. [关键数据结构](#11-关键数据结构)
12. [关键代码位置索引](#12-关键代码位置索引)

---

## 1. OS启动 - ACPI子系统初始化

### 调用流程

```
系统启动
    ↓
acpi_init()                    [drivers/acpi/bus.c]
    ↓
acpi_scan_init()              [drivers/acpi/scan.c:2697]
    ├─ acpi_pci_root_init()      [注册PCI Root Bridge处理器]
    ├─ acpi_platform_init()       [注册platform设备处理器]
    └─ acpi_scan_add_handler(&generic_device_handler)
    ↓
acpi_bus_scan(ACPI_ROOT_OBJECT)  [drivers/acpi/scan.c:2747]
```

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `acpi_scan_init()` | drivers/acpi/scan.c | 2697 |
| `acpi_pci_root_init()` | drivers/acpi/pci_root.c | - |
| `acpi_platform_init()` | drivers/acpi/acpi_platform.c | 193 |
| `acpi_bus_scan()` | drivers/acpi/scan.c | 2588 |

### 说明

- `acpi_scan_init()` 在系统启动时被调用，初始化ACPI扫描子系统
- 注册各种ACPI设备的扫描处理器（handler）
- 开始遍历ACPI命名空间，查找硬件设备

---

## 2. ACPI命名空间扫描

### 调用流程

```
acpi_bus_scan(ACPI_ROOT_OBJECT)  [scan.c:2588]
    ↓
acpi_bus_check_add()            [检查设备是否可添加]
    ↓
acpi_walk_namespace()            [遍历ACPI命名空间]
    ↓
acpi_bus_attach(device)          [scan.c:2273]
    ↓
acpi_scan_attach_handler(device)  [scan.c:2244]
    └─ 匹配ACPI设备的HID与注册的handler
```

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `acpi_bus_scan()` | drivers/acpi/scan.c | 2588 |
| `acpi_bus_attach()` | drivers/acpi/scan.c | 2273 |
| `acpi_scan_attach_handler()` | drivers/acpi/scan.c | 2244 |

### 说明

- 遍历ACPI命名空间中的所有设备
- 对每个设备尝试匹配已注册的handler
- Handler通过HID (Hardware ID)进行匹配

---

## 3. 发现PCI Root Bridge

当ACPI扫描发现 **PNP0A03** (PCI Root Bridge) 或 **PNP0A08** (PCIe Root Bridge) 时：

### ACPI设备标识

| HID | 描述 |
|-----|------|
| `PNP0A03` | PCI Root Bridge |
| `PNP0A08` | PCIe Root Bridge |
| `ACPI0016` | CXL Root Bridge |

### 调用流程

```
ACPI设备: PNP0A03 (PCI Root Bridge)
    ↓
匹配到 pci_root_handler  [drivers/acpi/pci_root.c:49]
    ├─ .ids = {"PNP0A03", 0}
    └─ .attach = acpi_pci_root_add
    ↓
调用 acpi_pci_root_add()  [drivers/acpi/pci_root.c:639]
```

### 关键代码位置

| 结构 | 文件 | 行号 |
|------|------|------|
| `pci_root_handler` | drivers/acpi/pci_root.c | 49 |
| `acpi_pci_root_add()` | drivers/acpi/pci_root.c | 639 |

---

## 4. acpi_pci_root_add() - 创建PCI Root

### 调用流程

```
acpi_pci_root_add(device)  [pci_root.c:639]
    ├─ 评估 _SEG (Segment Number)
    ├─ 评估 _CRS/_BBN (Bus Number Range)
    ├─ 创建 struct acpi_pci_root
    ├─ 协商OSC (Operating System Capabilities)
    │   └─ negotiate_os_control()
    │       ├─ _OSC方法协商ASPM、MSI、AER等支持
    │       └─ 请求PCIe特性控制权
    └─ pci_acpi_scan_root(root)  [pci_root.c:728]
```

### OSC协商的PCIe特性

| 特性 | 位 | 描述 |
|------|-----|------|
| Extended Config | 0x00000001 | 扩展配置支持 |
| ASPM | 0x00000002 | Active State Power Management |
| Clock PM | 0x00000004 | 时钟电源管理 |
| MSI | 0x00000008 | Message Signaled Interrupts |
| PCIe Hotplug | 0x00010000 | PCIe热插拔 |
| PME | 0x00020000 | Power Management Events |
| AER | 0x00040000 | Advanced Error Reporting |

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `acpi_pci_root_add()` | drivers/acpi/pci_root.c | 639 |
| `negotiate_os_control()` | drivers/acpi/pci_root.c | ~500 |
| `pci_acpi_scan_root()` | drivers/acpi/pci_root.c | 728 |

---

## 5. pci_acpi_scan_root() - 扫描PCI Root

### 调用流程

```
pci_acpi_scan_root(root)  [drivers/pci/pci-acpi.c:1664]
    ├─ 从ACPI获取资源信息（_CRS）
    ├─ 检查MCFG表（PCI Memory Mapped Configuration Space）
    ├─ 创建 pci_host_bridge
    ├─ 设置配置空间访问操作（pci_ops）
    └─ acpi_pci_root_create()
        └─ pci_scan_root_bus()  [drivers/pci/probe.c]
```

### MCFG表

MCFG (Memory Mapped Configuration Space) 表提供了PCIe配置空间的物理地址映射信息。

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `pci_acpi_scan_root()` | drivers/pci/pci-acpi.c | 1664 |
| `acpi_pci_root_create()` | drivers/pci/pci-acpi.c | ~1600 |
| `pci_scan_root_bus()` | drivers/pci/probe.c | 3439 |

---

## 6. pci_scan_root_bus() - 扫描PCI总线

### 调用流程

```
pci_scan_root_bus()  [drivers/pci/probe.c:3439]
    ├─ 创建 struct pci_bus
    ├─ 设置总线号范围
    ├─ pci_scan_child_bus()  [递归扫描子总线]
    │   ├─ pci_scan_slot()  [扫描每个slot]
    │   │   └─ pci_scan_single_device()  [扫描单个设备]
    │   │       ├─ pci_read_config_*()  [读取配置空间]
    │   │       ├─ pci_setup_device()  [probe.c:2018]
    │   │       │   ├─ 读取设备ID、类代码
    │   │       │   ├─ 读取BAR（Base Address Register）
    │   │       │   ├─ 读取能力（Capabilities）
    │   │       │   └─ pci_device_add()  [添加到设备列表]
    │   │       └─ pci_device_probe()  [探测设备驱动]
    │   └─ 处理PCI桥接设备
    └─ 返回 pci_bus
```

### 设备扫描详细流程

```
pci_scan_single_device(bus, devfn)  [probe.c:2810]
    ├─ pci_read_config_word(dev, PCI_VENDOR_ID, &l)
    ├─ pci_read_config_word(dev, PCI_DEVICE_ID, &h)
    ├─ 如果设备存在（vendor_id != 0xFFFF）
    │   └─ pci_setup_device(dev)  [probe.c:2018]
    │       ├─ 读取Header Type
    │       ├─ 设置设备类代码
    │       ├─ pci_read_bases()  [读取BAR]
    │       ├─ pci_find_capability()  [查找能力]
    │       └─ pci_device_add(dev, bus)  [probe.c:2758]
    │           └─ device_add(&dev->dev)  [添加到设备模型]
```

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `pci_scan_root_bus()` | drivers/pci/probe.c | 3439 |
| `pci_scan_child_bus()` | drivers/pci/probe.c | 3237 |
| `pci_scan_single_device()` | drivers/pci/probe.c | 2810 |
| `pci_setup_device()` | drivers/pci/probe.c | 2018 |
| `pci_device_add()` | drivers/pci/probe.c | 2758 |

---

## 7. Platform Device方式（某些架构）

对于某些ARM64或特定架构，PCIe RC可能通过platform device方式初始化：

### 调用流程

```
acpi_bus_attach()
    ↓
acpi_generic_device_attach()  [drivers/acpi/scan.c]
    ↓
acpi_create_platform_device()  [drivers/acpi/acpi_platform.c:110]
    ├─ 解析ACPI资源（_CRS）
    ├─ 创建 platform_device
    └─ platform_device_register_full()
        ↓
platform_device_register()
        ↓
device_attach(&pdev->dev)  [drivers/base/dd.c]
        ↓
platform_probe()  [drivers/base/platform.c:1420]
        ↓
platform_driver->probe(pdev)  [调用platform驱动的probe]
```

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `acpi_create_platform_device()` | drivers/acpi/acpi_platform.c | 110 |
| `platform_device_register()` | drivers/base/platform.c | 816 |
| `platform_probe()` | drivers/base/platform.c | 1420 |

---

## 8. Platform Driver Probe

### 调用流程

```
platformplatform_driver->probe(pdev)  [例如 pci-host-generic.c]
    ↓
pci_host_common_probe()  [drivers/pci/controller/pci-host-common.c:81]
    ├─ 从device tree获取匹配数据（pci_ecam_ops）
    ├─ devm_pci_alloc_host_bridge()  [分配pci_host_bridge]
    └─ pci_host_common_init(pdev, bridge, ops)
        ├─ pci_host_common_ecam_create()  [创建ECAM配置窗口]
        │   ├─ of_address_to_resource()  [获取配置空间物理地址]
        │   ├─ pci_ecam_create()  [映射配置空间]
        │   └─ 设置 bridge->sysdata 和 bridge->ops
        └─ pci_host_probe(bridge)  [drivers/pci/probe.c:292]
```

### 通用PCI主机控制器

| 驱动 | 文件 | 描述 |
|------|------|------|
| pci-host-generic | controller/pci-host-generic.c | 通用ECAM主机控制器 |
| pcie-brcmstb | controller/pcie-brcmstb.c | Broadcom STB PCIe |
| pcie-mediatek | controller/pcie-mediatek.c | MediaTek PCIe |
| pcie-rockchip | controller/pcie-rockchip-host.c | Rockchip PCIe |
| pcie-rcar | controller/pcie-rcar-host.c | Renesas R-Car PCIe |

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `pci_host_common_probe()` | drivers/pci/controller/pci-host-common.c | 81 |
| `pci_host_common_init()` | drivers/pci/controller/pci-host-common.c | 55 |
| `pci_host_common_ecam_create()` | drivers/pci/controller/pci-host-common.c | 25 |

---

## 9. pci_host_probe() - 最终初始化

### 调用流程

```
pci_host_probe(bridge)  [drivers/pci/probe.c:3292]
    ├─ 设置PCI主机桥的名称
    ├─ pci_scan_root_bus_bridge(bridge)  [probe.c:3399]
    │   ├─ pci_register_host_bridge(bridge)
    │   └─ pci_scan_child_bus(bridge->bus)
    ├─ pci_bus_claim_resources(bus)  [如果preserve_config]
    ├─ pci_assign_unassigned_root_bus_resources(bus)  [分配未分配的资源]
    ├─ pcie_bus_configure_settings()  [配置PCIe总线设置]
    ├─ pci_bus_add_devices(bridge->bus)  [添加设备到内核]
    │   └─ device_attach(&dev->dev)  [附加每个设备]
    └─ 设置电源管理
```

### 资源分配流程

```
pci_assign_unassigned_root_bus_resources(bus)
    ├─ pci_bus_assign_resources(bus)  [分配总线资源]
    │   ├─ pbus_size_mem()  [计算内存资源大小]
    │   ├─ pbus_size_io()   [计算IO资源大小]
    │   └─ pbus_assign_resources()  [实际分配]
    └─ pci_enable_bridges()  [启用桥接]
```

### 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `pci_host_probe()` | drivers/pci/probe.c | 3292 |
| `pci_scan_root_bus_bridge()` | drivers/pci/probe.c | 3399 |
| `pci_bus_add_devices()` | drivers/pci/bus.c | ~350 |
| `pci_assign_unassigned_root_bus_resources()` | drivers/pci/setup-bus.c | - |

---

## 10. 完整调用链总结

### ACPI方式完整调用链

```
系统启动
  └─ acpi_scan_init()
        └─ acpi_bus_scan(ACPI_ROOT_OBJECT)
              └─ acpi_bus_attach(device)
                    └─ acpi_scan_attach_handler()
                          └─ 匹配到 pci_root_handler
                                └─ acpi_pci_root_add()
                                      ├─ 协商OSC
                                      └─ pci_acpi_scan_root()
                                            └─ acpi_pci_root_create()
                                                  └─ pci_scan_root_bus()
                                                        └─ pci_scan_child_bus()
                                                              └─ pci_scan_single_device()
                                                                    ├─ pci_setup_device()
                                                                    └─ pci_device_add()
```

### Platform Device方式完整调用链

```
acpi_bus_attach()
  └─ acpi_create_platform_device()
        └─ platform_device_register()
              └─ device_attach()
                    └─ platform_probe()
                          └─ pci_host_common_probe()
                                └─ pci_host_common_init()
                                      └─ pci_host_probe()
                                            ├─ pci_scan_root_bus_bridge()
                                            └─ pci_bus_add_devices()
```

---

## 11. 关键数据结构

### ACPI相关结构

| 结构体 | 定义位置 | 用途 |
|--------|----------|------|
| `acpi_device` | drivers/acpi/internal.h | ACPI设备对象 |
| `acpi_pci_root` | drivers/acpi/pci_root.c | ACPI PCI Root桥信息 |
| `acpi_pci_root_ops` | drivers/pci/pci-acpi.c | ACPI PCI Root操作 |

### PCI核心结构

| 结构体 | 定义位置 | 用途 |
|--------|----------|------|
| `pci_host_bridge` | include/linux/pci.h | PCI主机桥 |
| `pci_bus` | include/linux/pci.h | PCI总线 |
| `pci_dev` | include/linux/pci.h | PCI设备 |
| `pci_ops` | include/linux/pci.h | PCI配置空间操作 |

### Platform相关结构

| 结构体 | 定义位置 | 用途 |
|--------|----------|------|
| `platform_device` | include/linux/platform_device.h | Platform设备 |
| `platform_driver` | include/linux/platform_device.h | Platform驱动 |
| `pci_ecam_ops` | include/linux/pci-ecam.h | ECAM操作 |

---

## 12. 关键代码位置索引

### ACPI核心

| 文件 | 关键函数 |
|------|----------|
| drivers/acpi/scan.c | `acpi_scan_init()`, `acpi_bus_scan()`, `acpi_bus_attach()` |
| drivers/acpi/pci_root.c | `acpi_pci_root_add()`, `negotiate_os_control()` |
| drivers/acpi/acpi_platform.c | `acpi_create_platform_device()` |

### PCI核心

| 文件 | 关键函数 |
|------|----------|
| drivers/pci/probe.c | `pci_scan_root_bus()`, `pci_scan_child_bus()`, `pci_scan_single_device()`, `pci_setup_device()`, `pci_device_add()`, `pci_host_probe()` |
| drivers/pci/bus.c | `pci_bus_add_devices()` |
| drivers/pci/setup-bus.c | `pci_assign_unassigned_root_bus_resources()` |
| drivers/pci/pci-acpi.c | `pci_acpi_scan_root()`, `acpi_pci_root_create()` |
| drivers/pci/pci-driver.c | `pci_device_probe()` |

### Platform总线

| 文件 | 关键函数 |
|------|----------|
| drivers/base/platform.c | `platform_probe()`, `platform_device_register()` |

### PCI控制器驱动

| 文件 | 关键函数 |
|------|----------|
| drivers/pci/controller/pci-host-common.c | `pci_host_common_probe()`, `pci_host_common_init()` |
| drivers/pci/controller/pci-host-generic.c | `gen_pci_driver.probe` |

---

## 附录

### A. ACPI相关方法

| 方法 | 描述 |
|------|------|
| `_SEG` | Segment Number |
| `_BBN` | Bus Base Number |
| `_CRS` | Current Resource Settings |
| `_OSC` | Operating System Capabilities |
| `_CBA` | Configuration Base Address |
| `_UID` | Unique ID |

### B. PCIe配置空间

| 寄存 | 偏移 | 描述 |
|------|------|------|
| Vendor ID | 0x00 | 厂商ID |
| Device ID | 0x02 | 设备ID |
| Class Code | 0x08-0x0B | 类代码 |
| BAR0-5 | 0x10-0x24 | Base Address Register |
| Command | 0x04 | 命令寄存器 |

### C. 参考文档

- PCI Firmware Specification: https://members.pcisig.com/
- ACPI Specification: https://uefi.org/specifications
- Linux Kernel Documentation: Documentation/PCI/

---

**文档版本**: 1.0  
**最后更新**: 2026-03-23  
**内核版本**: Linux 6.19.0