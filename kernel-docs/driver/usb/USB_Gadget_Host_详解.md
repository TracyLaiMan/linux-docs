# Linux USB Gadget Device 和 Host 详解

## 目录
1. [基本概念](#基本概念)
2. [架构层次](#架构层次)
3. [关键组件](#关键组件)
4. [工作流程](#工作流程)
5. [实际应用示例](#实际应用示例)
6. [主要区别总结](#主要区别总结)
7. [OTG 支持](#otg-支持)
8. [项目中的实现](#项目中的实现)

---

## 基本概念

### USB Host（USB 主机）

**定义**：USB Host 是 USB 总线的主控端，负责控制和管理整个 USB 总线。

**主要特点**：
- **角色**：主控端（Master），控制 USB 总线
- **功能**：枚举设备、分配地址、管理数据传输
- **典型场景**：PC、服务器、嵌入式设备作为主机连接外设
- **连接器类型**：A 型接口（Type-A）

**工作原理**：
- 初始化 USB 控制器硬件
- 检测连接的 USB 设备
- 为设备分配地址（1-127）
- 加载相应的设备驱动
- 管理数据传输和电源

### USB Gadget Device（USB 设备/外设）

**定义**：USB Gadget Device 是 USB 总线的从设备端，响应主机的控制请求并提供特定功能。

**主要特点**：
- **角色**：从设备端（Slave），被主机控制
- **功能**：响应主机请求，提供功能（如存储、串口、网络）
- **典型场景**：手机、U盘、摄像头等作为设备连接到主机
- **连接器类型**：B 型接口（Type-B）

**工作原理**：
- 初始化 USB 设备控制器硬件
- 注册 Gadget 功能驱动
- 响应主机的枚举请求
- 提供设备描述符和配置信息
- 处理主机的数据传输请求

---

## 架构层次

### USB Host 架构

```
┌─────────────────────────────────────┐
│        应用层 (Application)         │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│   USB 设备类驱动 (Class Driver)      │
│   - USB Storage                     │
│   - USB Serial                      │
│   - USB HID                         │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│      USB 核心层 (USB Core)          │
│   - 设备管理                        │
│   - 端点管理                        │
│   - 配置管理                        │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│   USB 主机控制器驱动 (HCD)           │
│   - EHCI (USB 2.0 High Speed)       │
│   - OHCI/UHCI (USB 1.1)             │
│   - xHCI (USB 3.0 SuperSpeed)       │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│      硬件控制器 (Hardware)           │
└─────────────────────────────────────┘
```

**各层说明**：

1. **应用层**：用户空间应用程序，如文件管理器、串口工具等
2. **USB 设备类驱动**：实现特定 USB 设备类的功能，如大容量存储、串口通信等
3. **USB 核心层**：提供统一的 USB 设备管理、端点管理、配置管理等核心功能
4. **HCD (Host Controller Driver)**：与硬件控制器交互的底层驱动
5. **硬件控制器**：实际的 USB 主机控制器硬件

### USB Gadget Device 架构

```
┌─────────────────────────────────────┐
│   应用层/用户空间 (Userspace)        │
│   - ConfigFS                        │
│   - SysFS                           │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│   Gadget 驱动 (Gadget Driver)        │
│   - g_mass_storage                  │
│   - g_serial                        │
│   - g_ether                         │
│   - g_hid                           │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│   Gadget API 层 (Gadget Framework)   │
│   - Composite Framework              │
│   - ConfigFS Support                │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│   USB 设备控制器驱动 (UDC)            │
│   - DWC2/DWC3                       │
│   - MUSB                            │
│   - ChipIdea                        │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│   硬件控制器 (Peripheral Controller)  │
└─────────────────────────────────────┘
```

**各层说明**：

1. **应用层/用户空间**：通过 ConfigFS 或 SysFS 配置 USB 功能
2. **Gadget 驱动**：实现具体的 USB 功能，如存储、串口、网络等
3. **Gadget API 层**：提供统一的 Gadget 框架，支持复合设备
4. **UDC (USB Device Controller)**：与硬件控制器交互的底层驱动
5. **硬件控制器**：实际的 USB 设备控制器硬件

---

## 关键组件

### USB Host 关键组件

#### 1. HCD (Host Controller Driver)

**作用**：与 USB 主机控制器硬件交互的底层驱动

**常见类型**：

| HCD 类型 | USB 版本 | 速度 | 说明 |
|---------|---------|------|------|
| **EHCI** | USB 2.0 | 480 Mbps | Enhanced Host Controller Interface |
| **OHCI** | USB 1.1 | 12 Mbps | Open Host Controller Interface |
| **UHCI** | USB 1.1 | 12 Mbps | Universal Host Controller Interface |
| **xHCI** | USB 3.0 | 5 Gbps | eXtensible Host Controller Interface |

**项目中的实现**：
- 位置：`kernel/t32-kernel-3.10.14/drivers/usb/host/`
- 支持多种 HCD 驱动：EHCI、OHCI、UHCI、xHCI 等

#### 2. USB Core

**作用**：提供统一的 USB 设备管理框架

**主要功能**：
- 设备枚举和识别
- 端点管理
- 配置管理
- 电源管理
- 设备驱动匹配

#### 3. USB Class Driver

**作用**：实现特定 USB 设备类的功能

**常见类型**：
- USB Storage：大容量存储设备
- USB Serial：串口设备
- USB HID：人机接口设备
- USB Audio：音频设备
- USB Video：视频设备

### USB Gadget Device 关键组件

#### 1. UDC (USB Device Controller)

**作用**：与 USB 设备控制器硬件交互的底层驱动

**主要功能**：
- 初始化硬件控制器
- 管理端点（Endpoint）
- 处理 USB 请求
- 管理电源状态

**数据结构**（来自项目代码）：

```c
struct usb_udc {
    struct usb_gadget_driver    *driver;    // Gadget 驱动指针
    struct usb_gadget           *gadget;    // Gadget 对象
    struct device               dev;        // 设备对象
    struct list_head            list;       // 链表节点
    // ... 其他字段
};
```

**项目中的 UDC 驱动**：
- DWC2/DWC3：DesignWare USB 控制器
- MUSB：Multipoint USB 控制器
- ChipIdea：ChipIdea USB 控制器
- 其他平台特定控制器

#### 2. Gadget Driver（功能驱动）

**作用**：实现具体的 USB 功能

**常见 Gadget 驱动**：

| Gadget 驱动 | 功能 | USB 类 |
|------------|------|--------|
| **g_mass_storage** | 大容量存储 | Mass Storage |
| **g_serial** | 串口通信 | CDC ACM |
| **g_ether** | 网络功能 | CDC ECM/NCM |
| **g_hid** | HID 设备 | HID |
| **g_audio** | 音频设备 | Audio |
| **g_webcam** | 摄像头 | Video |
| **g_printer** | 打印机 | Printer |
| **g_multi** | 复合设备 | Multiple |

**项目中的实现**：
- 位置：`kernel/t32-kernel-3.10.14/drivers/usb/gadget/`
- 支持多种 Gadget 驱动

#### 3. Composite Framework

**作用**：支持在一个 USB 设备上同时提供多个功能

**特点**：
- 一个设备可以同时提供多个接口
- 例如：同时提供存储和串口功能
- 通过 ConfigFS 动态配置

---

## 工作流程

### USB Host 工作流程

```
1. 系统启动
   ↓
2. HCD 初始化硬件控制器
   ↓
3. USB Core 初始化
   ↓
4. 检测 USB 设备插入
   ↓
5. 设备枚举（获取设备描述符）
   ↓
6. 分配设备地址（1-127）
   ↓
7. 获取配置描述符
   ↓
8. 加载匹配的设备驱动
   ↓
9. 设备就绪，开始数据传输
```

**详细步骤**：

1. **初始化阶段**：
   - HCD 驱动初始化硬件控制器
   - 配置中断和 DMA
   - 启动根集线器（Root Hub）

2. **设备检测**：
   - 硬件检测到设备插入
   - 触发中断或轮询检测

3. **设备枚举**：
   - 获取设备描述符（Device Descriptor）
   - 获取配置描述符（Configuration Descriptor）
   - 获取接口描述符（Interface Descriptor）
   - 获取端点描述符（Endpoint Descriptor）

4. **驱动匹配**：
   - 根据设备类、厂商ID、产品ID 匹配驱动
   - 加载相应的设备驱动

5. **数据传输**：
   - 通过端点进行数据传输
   - 支持控制、批量、中断、等时传输

### USB Gadget Device 工作流程

```
1. 系统启动
   ↓
2. UDC 初始化硬件控制器
   ↓
3. 注册 Gadget 驱动
   ↓
4. 连接到主机（物理连接）
   ↓
5. 主机检测到设备
   ↓
6. 响应主机枚举请求
   - 提供设备描述符
   - 提供配置描述符
   - 提供字符串描述符
   ↓
7. 主机配置设备
   ↓
8. 设备就绪，处理主机请求
   ↓
9. 数据传输（根据功能驱动）
```

**详细步骤**：

1. **初始化阶段**：
   - UDC 驱动初始化硬件控制器
   - 配置端点和 FIFO
   - 准备中断处理

2. **驱动注册**：
   - Gadget 驱动注册到 UDC
   - 绑定到硬件控制器
   - 设置设备描述符

3. **连接阶段**：
   - 物理连接到主机
   - 硬件检测到连接
   - 触发连接事件

4. **枚举响应**：
   - 响应 GET_DESCRIPTOR 请求
   - 提供设备描述符
   - 提供配置和接口描述符
   - 提供字符串描述符

5. **配置阶段**：
   - 主机选择配置
   - 激活端点
   - 准备数据传输

6. **运行阶段**：
   - 处理主机的数据传输请求
   - 根据功能驱动执行相应操作
   - 处理控制请求

---

## 实际应用示例

### 示例 1：Fastboot Gadget（项目中的实现）

**位置**：`bootloader/t41_uboot/u-boot/drivers/usb/gadget/g_fastboot.c`

**功能**：实现 Android Fastboot 协议，用于刷机

**关键代码结构**：

```c
// 设备描述符
static struct usb_device_descriptor device_desc = {
    .bLength = sizeof device_desc,
    .bDescriptorType = USB_DT_DEVICE,
    .bcdUSB = __constant_cpu_to_le16(0x0200),
    .bDeviceClass = USB_CLASS_VENDOR_SPEC,
    .idVendor = __constant_cpu_to_le16(CONFIG_G_FASTBOOT_VENDOR_NUM),
    .idProduct = __constant_cpu_to_le16(CONFIG_G_FASTBOOT_PRODUCT_NUM),
    // ...
};

// 配置函数
static int g_fastboot_do_config(struct usb_configuration *c)
{
    // 添加 Fastboot 功能
    return fastboot_add(c);
}
```

**使用场景**：
- Android 设备进入 Fastboot 模式
- 通过 USB 连接 PC
- PC 端使用 fastboot 命令刷写固件

### 示例 2：Mass Storage Gadget

**功能**：将设备存储空间作为 USB 大容量存储设备

**使用场景**：
- 手机连接 PC 时作为 U盘使用
- 嵌入式设备提供文件传输功能

**配置方法**（使用 ConfigFS）：

```bash
# 加载模块
modprobe libcomposite
modprobe usb_f_mass_storage

# 创建配置
mkdir -p /sys/kernel/config/usb_gadget/g1
cd /sys/kernel/config/usb_gadget/g1

# 设置设备信息
echo "0x1d6b" > idVendor
echo "0x0104" > idProduct
echo "0x0100" > bcdDevice
echo "0x0200" > bcdUSB

# 创建配置
mkdir -p configs/c.1
mkdir -p functions/mass_storage.usb0

# 绑定功能
ln -s functions/mass_storage.usb0 configs/c.1/

# 绑定 UDC
echo "musb-hdrc.0" > UDC
```

### 示例 3：Serial Gadget

**功能**：提供 USB 串口功能

**使用场景**：
- 嵌入式设备调试
- 远程终端访问
- 数据通信

**配置方法**：

```bash
# 加载模块
modprobe g_serial

# 在主机端会看到 /dev/ttyACM0 或 /dev/ttyGS0
```

---

## 主要区别总结

| 特性 | USB Host | USB Gadget Device |
|------|----------|-------------------|
| **角色** | 主控端（Master） | 从设备端（Slave） |
| **连接器** | A 型接口（Type-A） | B 型接口（Type-B） |
| **功能** | 控制总线，枚举设备 | 响应请求，提供功能 |
| **驱动类型** | HCD (Host Controller Driver) | UDC (USB Device Controller) |
| **配置选项** | `CONFIG_USB` | `CONFIG_USB_GADGET` |
| **典型应用** | PC、服务器、嵌入式主机 | 手机、U盘、摄像头、嵌入式设备 |
| **设备数量** | 可连接多个设备（最多127个） | 只能连接到一个主机 |
| **电源** | 提供电源（5V） | 从主机获取电源 |
| **数据传输方向** | 主动发起传输 | 响应主机请求 |
| **驱动位置** | `drivers/usb/host/` | `drivers/usb/gadget/` |

### 独立性说明

**重要**：USB Host 和 USB Gadget Device 在 Linux 内核中是**完全独立**的：

- Gadget 支持**不依赖** Host 支持（`CONFIG_USB_GADGET` 不依赖 `CONFIG_USB`）
- 可以只编译 Host 支持
- 可以只编译 Gadget 支持
- 可以同时编译两者（用于 OTG 设备）

---

## OTG 支持

### 什么是 OTG？

**OTG (On-The-Go)**：允许设备既可以作为 Host，也可以作为 Device，并且可以在两者之间切换。

### OTG 的特点

1. **双角色支持**：
   - 同一硬件控制器支持 Host 和 Device 两种模式
   - 通过软件或硬件信号切换角色

2. **连接器**：
   - 使用 Mini-AB 或 Micro-AB 接口
   - 通过 ID 引脚判断角色

3. **应用场景**：
   - 手机连接 U盘（Host 模式）
   - 手机连接 PC（Device 模式）
   - 两个 OTG 设备直接通信

### 项目中的 OTG 支持

根据项目代码，系统支持 OTG：

```c
// 来自 Kconfig
// With help from a special transceiver and a "Mini-AB" jack, 
// systems with both kinds of controller can also support 
// "USB On-the-Go" (CONFIG_USB_OTG).
```

### OTG 工作流程

```
1. 检测连接
   ↓
2. 读取 ID 引脚状态
   ↓
3. 判断角色
   - ID = 0 (GND) → Device 模式
   - ID = 1 (浮空) → Host 模式
   ↓
4. 初始化相应模式
   ↓
5. 运行
```

---

## 项目中的实现

### 项目结构

```
VisonT/
├── kernel/
│   ├── t32-kernel-3.10.14/
│   │   └── drivers/usb/
│   │       ├── host/          # USB Host 驱动
│   │       │   ├── ehci-hcd.c
│   │       │   ├── ohci-hcd.c
│   │       │   ├── xhci-hcd.c
│   │       │   └── ...
│   │       └── gadget/        # USB Gadget 驱动
│   │           ├── udc-core.c
│   │           ├── composite.c
│   │           ├── configfs.c
│   │           └── ...
│   └── t41-kernel-4.4.94/
│       └── drivers/usb/
│           └── ...
├── bootloader/
│   └── t41_uboot/
│       └── u-boot/
│           └── drivers/usb/
│               ├── host/      # U-Boot Host 支持
│               └── gadget/    # U-Boot Gadget 支持
│                   └── g_fastboot.c
└── ...
```

### 关键配置文件

#### 1. Kernel 配置

**Host 配置**：
- `CONFIG_USB=y`：启用 USB Host 支持
- `CONFIG_USB_EHCI_HCD=y`：EHCI 驱动
- `CONFIG_USB_OHCI_HCD=y`：OHCI 驱动
- `CONFIG_USB_XHCI_HCD=y`：xHCI 驱动

**Gadget 配置**：
- `CONFIG_USB_GADGET=y`：启用 USB Gadget 支持
- `CONFIG_USB_GADGET_DRIVERS_HAVE_BULK=y`：支持批量传输
- `CONFIG_USB_LIBCOMPOSITE=y`：复合设备支持
- `CONFIG_USB_CONFIGFS=y`：ConfigFS 支持
- `CONFIG_USB_G_MASS_STORAGE=y`：大容量存储
- `CONFIG_USB_G_SERIAL=y`：串口功能

#### 2. 设备树配置

USB Host 设备树示例：

```dts
usb0: usb@12340000 {
    compatible = "ingenic,usb";
    reg = <0x12340000 0x10000>;
    interrupts = <10>;
    // ...
};
```

USB Gadget 设备树示例：

```dts
usb_otg: usb@12350000 {
    compatible = "ingenic,usb-otg";
    reg = <0x12350000 0x10000>;
    interrupts = <11>;
    // ...
};
```

### 常用命令和工具

#### Host 模式

```bash
# 查看连接的 USB 设备
lsusb

# 查看 USB 设备树
lsusb -t

# 查看 USB 设备详细信息
cat /sys/kernel/debug/usb/devices

# 查看 USB 驱动
lsmod | grep usb
```

#### Gadget 模式

```bash
# 查看可用的 UDC
ls /sys/class/udc/

# 查看 Gadget 配置（如果使用 ConfigFS）
ls /sys/kernel/config/usb_gadget/

# 加载 Gadget 驱动
modprobe g_mass_storage
modprobe g_serial
```

---

## 总结

### 核心要点

1. **USB Host** 和 **USB Gadget Device** 是 USB 协议中的两个不同角色
2. 两者在 Linux 内核中**完全独立**实现
3. **Host** 控制总线，**Device** 提供功能
4. 通过 **OTG** 可以实现双角色支持
5. 项目同时支持 Host 和 Gadget 功能

### 选择建议

- **需要连接 USB 外设**（U盘、鼠标、键盘等）→ 使用 **USB Host**
- **需要作为 USB 设备**（被 PC 识别、文件传输等）→ 使用 **USB Gadget Device**
- **需要两种功能**（如手机）→ 使用 **OTG** 支持

### 参考资料

- Linux USB 官方文档：`Documentation/usb/`
- USB 规范：USB 2.0/3.0 Specification
- 项目代码：`kernel/*/drivers/usb/`

