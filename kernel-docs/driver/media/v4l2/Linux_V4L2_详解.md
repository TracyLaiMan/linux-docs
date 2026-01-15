# Linux V4L2 详解

## 目录
1. [基本概念](#基本概念)
2. [V4L2 架构](#v4l2-架构)
3. [核心数据结构](#核心数据结构)
4. [驱动开发框架](#驱动开发框架)
5. [缓冲区管理](#缓冲区管理)
6. [IOCTL 接口](#ioctl-接口)
7. [子设备 (Subdev)](#子设备-subdev)
8. [实际应用示例](#实际应用示例)
9. [用户空间编程](#用户空间编程)
10. [项目中的实现](#项目中的实现)

---

## 基本概念

### 什么是 V4L2？

**V4L2 (Video4Linux2)** 是 Linux 内核中用于视频设备的标准 API 框架，用于支持：

- **视频采集设备**：摄像头、视频采集卡
- **视频输出设备**：视频编码器、显示器
- **视频处理设备**：编解码器、图像处理单元
- **VBI (Vertical Blanking Interval)**：垂直消隐间隔数据
- **Radio**：收音机设备

### V4L2 的特点

1. **统一的 API**：提供统一的接口访问各种视频设备
2. **多设备支持**：一个物理设备可以导出多个设备节点
3. **子设备架构**：支持复杂的多芯片视频系统
4. **缓冲区管理**：提供高效的视频缓冲区管理机制
5. **格式支持**：支持多种像素格式和视频标准

### V4L2 vs V4L1

| 特性 | V4L1 | V4L2 |
|------|------|------|
| **状态** | 已废弃 | 当前标准 |
| **API 设计** | 简单但有限 | 功能强大且灵活 |
| **缓冲区管理** | 基础支持 | 完善的缓冲区框架 |
| **格式支持** | 有限 | 广泛支持 |
| **多设备** | 不支持 | 支持 |

---

## V4L2 架构

### 整体架构图

```
┌─────────────────────────────────────────┐
│        用户空间应用程序                    │
│  (ffmpeg, v4l2-utils, 自定义应用)        │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         V4L2 用户空间 API                 │
│  (ioctl, read, mmap, poll)               │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         V4L2 核心层                       │
│  - v4l2-dev.c (设备节点管理)              │
│  - v4l2-ioctl.c (IOCTL 处理)             │
│  - v4l2-device.c (设备管理)               │
│  - v4l2-subdev.c (子设备管理)            │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│       Video Device 驱动层                 │
│  - video_device (设备节点)               │
│  - v4l2_ioctl_ops (IOCTL 操作)           │
│  - v4l2_file_operations (文件操作)       │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│       缓冲区管理 (Videobuf2)              │
│  - vb2_queue (缓冲区队列)                 │
│  - vb2_buffer (缓冲区)                    │
│  - vb2_ops (驱动回调)                    │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│        硬件驱动层                         │
│  - 摄像头驱动                             │
│  - 视频采集卡驱动                         │
│  - 编解码器驱动                           │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│           硬件设备                         │
└─────────────────────────────────────────┘
```

### 设备节点

V4L2 设备在 `/dev` 目录下创建字符设备节点：

- `/dev/videoX`：视频采集/输出设备（X = 0, 1, 2, ...）
- `/dev/vbiX`：VBI 设备
- `/dev/radioX`：收音机设备
- `/dev/v4l-subdevX`：子设备节点（可选）

### 设备类型

```c
// 来自 include/media/v4l2-dev.h
#define VFL_TYPE_GRABBER    0  // 视频采集/输出
#define VFL_TYPE_VBI        1  // VBI 设备
#define VFL_TYPE_RADIO      2  // 收音机
#define VFL_TYPE_SUBDEV     3  // 子设备
```

---

## 核心数据结构

### 1. v4l2_device

**作用**：表示一个 V4L2 设备实例，管理整个设备及其子设备。

**定义**（来自项目代码）：

```c
struct v4l2_device {
    struct device *dev;              // 父设备
    struct media_device *mdev;       // 媒体设备（可选）
    struct list_head subdevs;        // 子设备链表
    spinlock_t lock;                // 锁
    char name[V4L2_DEVICE_NAME_SIZE]; // 设备名称
    void (*notify)(struct v4l2_subdev *sd,
                   unsigned int notification, void *arg); // 通知回调
    struct v4l2_ctrl_handler *ctrl_handler; // 控制处理器
    struct v4l2_prio_state prio;      // 优先级状态
    struct mutex ioctl_lock;         // IOCTL 锁
    struct kref ref;                 // 引用计数
    void (*release)(struct v4l2_device *v4l2_dev); // 释放回调
};
```

**关键函数**：

```c
// 注册设备
int v4l2_device_register(struct device *dev, struct v4l2_device *v4l2_dev);

// 注销设备
void v4l2_device_unregister(struct v4l2_device *v4l2_dev);

// 断开连接（热插拔设备）
void v4l2_device_disconnect(struct v4l2_device *v4l2_dev);
```

### 2. video_device

**作用**：表示一个 V4L2 设备节点，对应 `/dev/videoX`。

**定义**（来自项目代码）：

```c
struct video_device {
#if defined(CONFIG_MEDIA_CONTROLLER)
    struct media_entity entity;      // 媒体实体
#endif
    const struct v4l2_file_operations *fops; // 文件操作
    struct device dev;                // 设备对象
    struct cdev *cdev;                // 字符设备
    struct device *parent;            // 父设备
    struct v4l2_device *v4l2_dev;     // V4L2 设备
    struct v4l2_ctrl_handler *ctrl_handler; // 控制处理器
    struct vb2_queue *queue;          // 缓冲区队列
    struct v4l2_prio_state *prio;     // 优先级状态
    
    char name[32];                    // 设备名称
    int vfl_type;                     // 设备类型
    int vfl_dir;                      // 方向（RX/TX/M2M）
    int minor;                        // 次设备号
    u16 num;                          // 设备编号
    unsigned long flags;              // 标志
    int index;                        // 索引
    
    spinlock_t fh_lock;               // 文件句柄锁
    struct list_head fh_list;         // 文件句柄链表
    
    int debug;                        // 调试级别
    
    v4l2_std_id tvnorms;              // 支持的电视标准
    v4l2_std_id current_norm;         // 当前电视标准
    
    void (*release)(struct video_device *vdev); // 释放回调
    const struct v4l2_ioctl_ops *ioctl_ops; // IOCTL 操作
    struct mutex *lock;                // 互斥锁
};
```

**关键函数**：

```c
// 注册视频设备
int video_register_device(struct video_device *vdev, int type, int nr);

// 注销视频设备
void video_unregister_device(struct video_device *vdev);

// 分配视频设备
struct video_device *video_device_alloc(void);

// 释放视频设备
void video_device_release(struct video_device *vdev);
```

### 3. v4l2_subdev

**作用**：表示一个子设备，通常是 I2C 设备（如传感器、编解码器）。

**特点**：
- 通过 I2C 总线连接
- 可以独立注册设备节点
- 通过 v4l2_device 管理

### 4. vb2_queue

**作用**：管理视频缓冲区队列，是 Videobuf2 框架的核心。

**定义**：

```c
struct vb2_queue {
    enum v4l2_buf_type type;          // 缓冲区类型
    unsigned int io_modes;            // IO 模式（MMAP/USERPTR/DMABUF）
    struct vb2_buffer *bufs[VB2_MAX_FRAME]; // 缓冲区数组
    unsigned int num_buffers;         // 缓冲区数量
    void *drv_priv;                    // 驱动私有数据
    const struct vb2_ops *ops;        // 驱动回调
    const struct vb2_mem_ops *mem_ops; // 内存操作
    // ...
};
```

---

## 驱动开发框架

### 驱动结构关系

```
设备实例 (Device Instance)
  |
  +-- v4l2_device (设备管理)
  |     |
  |     +-- v4l2_subdev[] (子设备数组)
  |
  +-- video_device (设备节点)
  |     |
  |     +-- v4l2_file_operations (文件操作)
  |     +-- v4l2_ioctl_ops (IOCTL 操作)
  |     +-- vb2_queue (缓冲区队列)
  |
  +-- 驱动私有数据
```

### 驱动开发步骤

#### 1. 定义驱动私有结构

```c
struct my_video_device {
    struct v4l2_device v4l2_dev;      // V4L2 设备
    struct video_device *vdev;        // 视频设备节点
    struct vb2_queue queue;           // 缓冲区队列
    struct mutex lock;                // 锁
    // ... 其他私有数据
};
```

#### 2. 初始化 v4l2_device

```c
static int my_video_probe(struct platform_device *pdev)
{
    struct my_video_device *my_dev;
    
    // 分配设备结构
    my_dev = kzalloc(sizeof(*my_dev), GFP_KERNEL);
    
    // 注册 v4l2_device
    ret = v4l2_device_register(&pdev->dev, &my_dev->v4l2_dev);
    if (ret < 0)
        goto err_free;
    
    // 设置设备名称
    v4l2_device_set_name(&my_dev->v4l2_dev, "my-video", &instance);
    
    // ...
}
```

#### 3. 创建 video_device

```c
// 分配 video_device
my_dev->vdev = video_device_alloc();
if (!my_dev->vdev) {
    ret = -ENOMEM;
    goto err_v4l2_unregister;
}

// 设置 video_device
my_dev->vdev->release = video_device_release;
my_dev->vdev->fops = &my_video_fops;
my_dev->vdev->ioctl_ops = &my_video_ioctl_ops;
my_dev->vdev->v4l2_dev = &my_dev->v4l2_dev;
my_dev->vdev->queue = &my_dev->queue;
snprintf(my_dev->vdev->name, sizeof(my_dev->vdev->name),
         "My Video Device");

// 注册 video_device
ret = video_register_device(my_dev->vdev, VFL_TYPE_GRABBER, -1);
if (ret < 0)
    goto err_video_release;
```

#### 4. 实现文件操作

```c
static const struct v4l2_file_operations my_video_fops = {
    .owner = THIS_MODULE,
    .open = my_video_open,
    .release = my_video_release,
    .read = my_video_read,
    .poll = vb2_fop_poll,
    .mmap = vb2_fop_mmap,
    .unlocked_ioctl = video_ioctl2,
};
```

#### 5. 实现 IOCTL 操作

```c
static const struct v4l2_ioctl_ops my_video_ioctl_ops = {
    .vidioc_querycap = my_video_querycap,
    .vidioc_enum_fmt_vid_cap = my_video_enum_fmt,
    .vidioc_g_fmt_vid_cap = my_video_g_fmt,
    .vidioc_s_fmt_vid_cap = my_video_s_fmt,
    .vidioc_reqbufs = vb2_ioctl_reqbufs,
    .vidioc_querybuf = vb2_ioctl_querybuf,
    .vidioc_qbuf = vb2_ioctl_qbuf,
    .vidioc_dqbuf = vb2_ioctl_dqbuf,
    .vidioc_streamon = vb2_ioctl_streamon,
    .vidioc_streamoff = vb2_ioctl_streamoff,
    // ...
};
```

---

## 缓冲区管理

### Videobuf2 框架

V4L2 使用 **Videobuf2** 框架管理视频缓冲区，提供：

- **多种内存类型**：MMAP、USERPTR、DMABUF
- **多平面支持**：单平面、多平面（如 YUV420）
- **DMA 支持**：DMA 连续、DMA 分散/聚集
- **零拷贝**：支持零拷贝操作

### 缓冲区类型

```c
enum v4l2_memory {
    V4L2_MEMORY_MMAP = 1,      // 内存映射
    V4L2_MEMORY_USERPTR = 2,   // 用户指针
    V4L2_MEMORY_OVERLAY = 3,   // 覆盖（已废弃）
    V4L2_MEMORY_DMABUF = 4,    // DMA 缓冲区
};
```

### 缓冲区操作流程

```
1. VIDIOC_REQBUFS
   ↓
   分配缓冲区
   ↓
2. VIDIOC_QUERYBUF
   ↓
   查询缓冲区信息
   ↓
3. VIDIOC_QBUF
   ↓
   将缓冲区加入队列
   ↓
4. VIDIOC_STREAMON
   ↓
   开始流传输
   ↓
5. VIDIOC_DQBUF
   ↓
   从队列取出已填充的缓冲区
   ↓
6. 处理数据后再次 QBUF
   ↓
   循环步骤 5-6
   ↓
7. VIDIOC_STREAMOFF
   ↓
   停止流传输
```

### vb2_ops 回调

驱动需要实现以下回调：

```c
struct vb2_ops {
    int (*queue_setup)(struct vb2_queue *q,
                       unsigned int *num_buffers,
                       unsigned int *num_planes,
                       unsigned long sizes[],
                       void *alloc_ctxs[]);
    
    int (*buf_prepare)(struct vb2_buffer *vb);
    void (*buf_queue)(struct vb2_buffer *vb);
    int (*start_streaming)(struct vb2_queue *q, unsigned int count);
    void (*stop_streaming)(struct vb2_queue *q);
    void (*buf_finish)(struct vb2_buffer *vb);
    void (*buf_cleanup)(struct vb2_buffer *vb);
};
```

---

## IOCTL 接口

### 主要 IOCTL 命令

#### 1. 设备信息

```c
// 查询设备能力
VIDIOC_QUERYCAP

// 查询控制项
VIDIOC_QUERYCTRL
VIDIOC_QUERYMENU

// 获取/设置控制项
VIDIOC_G_CTRL
VIDIOC_S_CTRL
```

#### 2. 格式操作

```c
// 枚举支持的格式
VIDIOC_ENUM_FMT

// 获取/设置格式
VIDIOC_G_FMT
VIDIOC_S_FMT

// 尝试格式（不实际设置）
VIDIOC_TRY_FMT
```

#### 3. 缓冲区操作

```c
// 请求缓冲区
VIDIOC_REQBUFS

// 查询缓冲区
VIDIOC_QUERYBUF

// 入队/出队缓冲区
VIDIOC_QBUF
VIDIOC_DQBUF

// 开始/停止流
VIDIOC_STREAMON
VIDIOC_STREAMOFF
```

---

## 用户空间编程

### 基本流程

```c
#include <linux/videodev2.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int main()
{
    int fd;
    struct v4l2_capability cap;
    struct v4l2_format fmt;
    
    // 1. 打开设备
    fd = open("/dev/video0", O_RDWR);
    
    // 2. 查询设备能力
    ioctl(fd, VIDIOC_QUERYCAP, &cap);
    
    // 3. 设置格式
    memset(&fmt, 0, sizeof(fmt));
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    fmt.fmt.pix.width = 640;
    fmt.fmt.pix.height = 480;
    fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
    ioctl(fd, VIDIOC_S_FMT, &fmt);
    
    // 4. 请求缓冲区
    // 5. 映射缓冲区
    // 6. 开始流传输
    // 7. 采集帧
    // 8. 停止流传输
    
    close(fd);
    return 0;
}
```

### 常用工具

1. **v4l2-utils**：用户空间工具集
   - `v4l2-ctl`：控制 V4L2 设备
   - `v4l2-compliance`：测试 V4L2 驱动

2. **示例命令**：

```bash
# 列出设备
v4l2-ctl --list-devices

# 查询设备能力
v4l2-ctl --device=/dev/video0 --all

# 设置格式
v4l2-ctl --device=/dev/video0 --set-fmt-video=width=640,height=480,pixelformat=YUYV
```

---

## 项目中的实现

### 项目结构

```
VisonT/
├── kernel/
│   ├── t32-kernel-3.10.14/
│   │   └── drivers/media/
│   │       ├── v4l2-core/          # V4L2 核心
│   │       │   ├── v4l2-dev.c     # 设备节点管理
│   │       │   ├── v4l2-device.c  # 设备管理
│   │       │   ├── v4l2-ioctl.c   # IOCTL 处理
│   │       │   └── ...
│   │       ├── platform/          # 平台驱动
│   │       └── usb/               # USB 视频设备
│   └── t41-kernel-4.4.94/
│       └── drivers/media/
│           └── ...
└── ...
```

### 关键配置文件

#### Kernel 配置

```kconfig
# V4L2 核心支持
CONFIG_VIDEO_V4L2=y
CONFIG_VIDEO_DEV=y

# Videobuf2 支持
CONFIG_VIDEOBUF2_CORE=y
CONFIG_VIDEOBUF2_MEMOPS=y
CONFIG_VIDEOBUF2_VMALLOC=y
CONFIG_VIDEOBUF2_DMA_CONTIG=y
```

---

## 总结

### 核心要点

1. **V4L2 是 Linux 视频设备的标准框架**
2. **三层架构**：用户空间 API、V4L2 核心、驱动层
3. **关键组件**：v4l2_device、video_device、vb2_queue
4. **缓冲区管理**：使用 Videobuf2 框架
5. **子设备支持**：通过 v4l2_subdev 管理复杂设备

### 开发建议

1. **使用框架提供的函数**：不要直接操作底层结构
2. **正确实现回调**：特别是 vb2_ops 中的回调
3. **处理错误情况**：检查所有返回值
4. **测试驱动**：使用 v4l2-compliance 测试
5. **参考现有驱动**：学习项目中的实际实现

### 参考资料

- Linux 内核文档：`Documentation/video4linux/`
- V4L2 API 规范：`include/uapi/linux/videodev2.h`
- 项目代码：`kernel/*/drivers/media/v4l2-core/`
