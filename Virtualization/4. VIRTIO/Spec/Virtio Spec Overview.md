
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [摘要](#摘要)
- [Virtio Spec 协议简介](#virtio-spec-协议简介)
  - [virtio 设备的四要素](#virtio-设备的四要素)
    - [设备状态域](#设备状态域)
    - [feature bits](#feature-bits)
    - [virtqueue](#virtqueue)
  - [transport 选项](#transport-选项)
- [virtio-pci 呈现](#virtio-pci-呈现)
  - [virtio legacy](#virtio-legacy)
  - [virtio modern](#virtio-modern)
- [前后端数据共享](#前后端数据共享)
- [前后端通信机制 (irqfd 与 ioeventfd)](#前后端通信机制-irqfd-与-ioeventfd)
- [virtio 核心代码分析, 以 virtio-net 为例](#virtio-核心代码分析-以-virtio-net-为例)
  - [前后端握手流程](#前后端握手流程)
  - [virtio-net 网卡收发在 virtqueue 上的实现](#virtio-net-网卡收发在-virtqueue-上的实现)
    - [virtio-net 网卡发包](#virtio-net-网卡发包)
    - [virtio-net 网卡收包](#virtio-net-网卡收包)
- [reference](#reference)

<!-- /code_chunk_output -->

# 摘要

半虚拟化设备 (Virtio Device) 在当前云计算虚拟化场景下已经得到了非常广泛的应用, 并且现在也有越来越多的物理设备也开始支持 Virtio 协议, 即所谓的 Virtio Offload, 通过将 virtio 协议卸载到硬件上 (例如 virtio-net 网卡卸载, virtio-scsi 卸载) 让物理机和虚拟机都能够获得加速体验. 本文中我们来重点了解一下 virtio 技术中的一些关键点, 方便我们加深对半虚拟化的理解. 本文适合对 IO 虚拟化有一定了解的人群阅读, 本文的目的是对想要了解 virtio 内部机制的读者提供帮助.

在开始了解 virtio 之前, 我们先思考一下几个相关问题:

* virtio 设备有哪几种呈现方式?
* virtio-pci 设备的配置空间都有哪些内容?
* virtio 前端和后端基于共享内存机制进行通信, 它是凭什么可以做到无锁的?
* virtio 机制中有那几个关键的数据结构? virtio 配置接口存放在哪里? virtio 是如何工作的?
* virtio 前后端是如何进行通信的? irqfd 和 ioeventfd 是什么回事儿? 在 virtio 前后端通信中是怎么用到的?
* virtio 设备支持 MSIx, 在 qemu/kvm 中具体是怎么实现对 MSIx 的模拟呢?
* virtio modern 相对于 virtio legay 多了哪些新特性?

# Virtio Spec 协议简介

virtio 协议标准最早由 IBM 提出, virtio 作为一套标准协议现在有专门的技术委员会进行管理, 具体的标准可以访问 [virtio 官网](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html), 开发者可以向技术委员会提供新的 virtio 设备提案 (RFC), 经过委员会通过后可以增加新的 virtio 设备类型.

## virtio 设备的四要素

组成**一个 virtio 设备**的**四要素**包括: **设备状态域**, **feature bits**, **设备配置空间**, 一个或者多个 **virtqueue**.

### 设备状态域

其中**设备状态域**包含 6 种状态:

* `ACKNOWLEDGE` (1): GuestOS 发现了这个设备, 并且认为这是**一个有效的 virtio 设备**;
* `DRIVER` (2): GuestOS **知道该如何驱动这个设备**;
* `FAILED` (128): GuestOS **无法正常驱动这个设备**;
* `FEATURES_OK` (8): GuestOS **认识所有的 feature**, 并且 feature 协商一完成;
* `DRIVER_OK` (4): 驱动加载完成, **设备可以投入使用**了;
* `DEVICE_NEEDS_RESET` (64): **设备触发了错误**, 需要重置才能继续工作.

### feature bits

feature bits 用来标志设备支持**哪个特性**, 其中

* `bit 0 - bit 23` 是**特定设备**可以使用的 feature bits,
* `bit 24 - bit 37` 预给队列和 feature 协商机制,
* `bit 38` 以上保留给未来其他用途.

例如: 对于 virtio-net 设备而言, feature bit0 表示网卡设备支持 checksum 校验. `VIRTIO_F_VERSION_1` 这个 feature bit 用来表示设备是否支持 virtio 1.0 spec 标准.

### virtqueue

在 virtio 协议中, **所有的设备**都使用 **virtqueue** 来进行**数据的传输**. 每个设备可以有 0 个或者**多个 virtqueue**, **每个 virtqueue** 占用 **2 个或者更多个 4K 的物理页**.

virtqueue 有 **Split Virtqueues** 和 **Packed Virtqueues** 两种模式.

在 Split virtqueues 模式下 virtqueue 被分成**若干个部分**, 每个部分都是前端驱动或者后端**单向可写**的 (不能两端同时写). 每个 virtqueue 都有一个 16bit 的 queue size 参数, 表示队列的总长度. 每个 virtqueue 由 3 个部分组成:

```
+-------------------+--------------------------------+-----------------------+
| Descriptor Table  |   Available Ring  (padding)    |       Used Ring       |
+-------------------+--------------------------------+-----------------------+
```

* **Descriptor Table**: 存放 **IO 传输请求信息**;
* **Available Ring**: 记录了 Descriptor Table 表中的 I/O 请求下发信息, 前端 Driver 可写后端只读;
* **Used Ring**: 记录 Descriptor Table 表中**已被提交到硬件的信息**, 前端 Driver 只读后端可写.

整个 virtio 协议中设备 IO 请求的工作机制可以简单地概括为:

1. **前端驱动**将 **IO 请求**放到 **Descriptor Table** 中, 然后将**索引**更新到 **Available Ring** 中, 最后 **kick 后端**去取数据;
2. **后端**取出 IO 请求进行**处理**, 然后将**结果刷新到 Descriptor Table** 中再**更新 Using Ring**, 然后**发送中断 notify 前端**.

## transport 选项

从 virtio 协议可以了解到 virtio 设备支持 3 种设备呈现模式:

* Virtio Over PCI BUS, 依旧遵循 PCI 规范, **挂载到 PCI 总线**上, **作为 virtio-pci 设备**呈现;
* Virtio Over MMIO, 部分**不支持 PCI 协议**的虚拟化平台可以使用这种工作模式, **直接挂载到系统总线**上;
* Virtio Over Channel I/O: 主要用在 s390 平台上, virtio-ccw 使用这种基于 channel I/O 的机制.

其中, Virtio Over PCI BUS 的使用比较广泛, **作为 PCI 设备**需**按照规范**要**通过 PCI 配置空间**来向**操作系统**报告设备支持的**特性集合**, 这样操作系统才知道这是一个**什么类型**的 virtio 设备, 并调用**对应的前端驱动**和这个设备进行握手, 进而将设备驱动起来. 

**QEMU** 会给 virtio 设备**模拟 PCI 配置空间**, 对于 virtio 设备来说 **PCI Vendor ID** 固定为 `0x1AF4`, **PCI Device ID** 为 `0x1000` 到 `0x107F` 之间的是 virtio 设备. 同时, 在**不支持 PCI 协议**的虚拟化平台上, virtio 设备也可以直接通过 MMIO 进行呈现, [virtio-spec 4.2 Virtio Over MMIO](https://docs.oasis-open.org/virtio/virtio/v1.1/cs01/virtio-v1.1-cs01.html#x1-1440002) 有针对 virtio-mmio 设备呈现方式的详细描述, **mmio 相关信息**可以直接通过**内核参数**报告给 Linux 操作系统. 

本文主要基于 virtio-pci 展开讨论.

# virtio-pci 呈现

前面提到 virtio 设备有 feature bits, virtqueue 等四要素, 那么在 virtio-pci 模式下是如何呈现的呢? 从 virtio spec 来看, **老的 virtio 协议**和**新的 virtio 协议**在这一块有很大改动.

## virtio legacy

**virtio legacy** (virtio 0.95) 协议规定, 对应的**配置数据结构** (virtio common configuration structure) 应该存放在设备的 **BAR0** 里面, 我们称之为 virtio legacy interface, 其结构如下:

```
                       virtio legacy ==> Mapped into PCI BAR0
    +------------------------------------------------------------------+
    |                    Host Feature Bits [0: 31]                       |
    +------------------------------------------------------------------+
    |                    Guest Feature Bits [0: 31]                      |
    +------------------------------------------------------------------+
    |                    Virtqueue Address PFN                         |
    +---------------------------------+--------------------------------+
    |           Queue Select          |           Queue Size           |
    +----------------+----------------+--------------------------------+
    |   ISR Status   | Device Stat    |           Queue Notify         |
    +----------------+----------------+--------------------------------+
    |       MSI Config Vector         |         MSI Queue Vector       |
    +---------------------------------+--------------------------------+
```

## virtio modern

对于新的 virtio modern, 协议将配置结构划分为 **5 种类型**:

```cpp
/* Common configuration */
#define VIRTIO_PCI_CAP_COMMON_CFG        1
/* Notifications */
#define VIRTIO_PCI_CAP_NOTIFY_CFG        2
/* ISR Status */
#define VIRTIO_PCI_CAP_ISR_CFG           3
/* Device specific configuration */
#define VIRTIO_PCI_CAP_DEVICE_CFG        4
/* PCI configuration access */
#define VIRTIO_PCI_CAP_PCI_CFG           5
```

以上的每种配置结构是直接映射到 virtio 设备的 **BAR** 空间内, 那么如何指定**每种配置结构的位置**呢? 答案是通过 **PCI Capability list** 方式去指定, 这和物理 PCI 设备是一样的, 体现了 virtio-pci 的协议兼容性. 只是略微不同的是, virtio-pci 的 Capability 有一个**统一的结构**, 如下.

```cpp
struct virtio_pci_cap {
        u8 cap_vndr;    /* Generic PCI field: PCI_CAP_ID_VNDR */
        u8 cap_next;    /* Generic PCI field: next ptr. */
        u8 cap_len;     /* Generic PCI field: capability length */
        u8 cfg_type;    /* Identifies the structure. */
        u8 bar;         /* Where to find it. */
        u8 padding [3];  /* Pad to full dword. */
        le32 offset;    /* Offset within bar. */
        le32 length;    /* Length of the structure, in bytes. */
};
```

其中 `cfg_type` 表示 Cap 的**类型**, `bar` 表示这个配置结构被映射到的 **BAR 空间号**. 这样每个配置结构都可以**通过 BAR 空间直接访问**, 或者**通过 PCI 配置空间**的 `VIRTIO_PCI_CAP_PCI_CFG` 域进行访问. 每个 Cap 的具体结构定义可以参考 `virtio spec 4.1.4.3` 小节. 为了方便理解这里以一张 virtio-net 网卡为例:

```
[root@localhost ~]# lspci -vvvs 04: 00.0
04: 00.0 Ethernet controller: Red Hat, Inc. Virtio network device (rev 01)
    Subsystem: Red Hat, Inc. Device 1100
    Physical Slot: 0-1
    Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx+
    Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
    Latency: 0
    Interrupt: pin A routed to IRQ 21
    Region 1: Memory at fe840000 (32-bit, non-prefetchable) [size=4K]
    Region 4: Memory at fa600000 (64-bit, prefetchable) [size=16K]
    Expansion ROM at fe800000 [disabled] [size=256K]
    Capabilities: [dc] MSI-X: Enable+ Count=10 Masked-
        Vector table: BAR=1 offset=00000000
        PBA: BAR=1 offset=00000800
    Capabilities: [c8] Vendor Specific Information: VirtIO: <unknown>
        BAR=0 offset=00000000 size=00000000
    Capabilities: [b4] Vendor Specific Information: VirtIO: Notify
        BAR=4 offset=00003000 size=00001000 multiplier=00000004
    Capabilities: [a4] Vendor Specific Information: VirtIO: DeviceCfg
        BAR=4 offset=00002000 size=00001000
    Capabilities: [94] Vendor Specific Information: VirtIO: ISR
        BAR=4 offset=00001000 size=00001000
    Capabilities: [84] Vendor Specific Information: VirtIO: CommonCfg
        BAR=4 offset=00000000 size=00001000
```

MSI-X 的 vector table 和 PBA 放到了 BAR1 里面, BAR4 里放了 common cfg, 设备 isr 状态信息, device cfg, driver notify 信息等.

# 前后端数据共享

传统的纯模拟设备在工作的时候, 会触发**频繁的陷入陷出**, 而且 IO 请求的**内容**要进行**多次拷贝传递**, 严重影响了设备的 IO 性能. 

virtio 为了提升设备的 IO 性能, 采用了**共享内存机制**, **前端驱动**会提前申请好**一段物理地址空间**用来**存放 IO 请求**, 然后将这段地址的 **GPA** 告诉 **QEMU**.

**前端驱动**在**下发 IO 请求**后, **QEMU** 可以直接从共享内存中**取出请求**, 然后将完成后的**结果又直接写到虚拟机对应地址**上去. 整个过程中可以做到直投直取, 省去了不必要的数据拷贝开销.

Virtqueue 是整个 virtio 方案的灵魂所在. **每个 virtqueue** 都包含 **3 张表**, **Descriptor Table** 存放了 **IO 请求描述符**, **Available Ring** 记录了**当前哪些描述符是可用**的, **Used Ring** 记录了**哪些描述符已经被后端使用**了.

```
                          +------------------------------------+
                          |       virtio  guest driver         |
                          +-----------------+------------------+
                            /               |              ^
                           /                |               \
                          put            update             get
                         /                  |                 \
                        V                   V                  \
                   +----------+      +------------+        +----------+
                   |          |      |            |        |          |
                   +----------+      +------------+        +----------+
                   | available|      | descriptor |        |   used   |
                   |   ring   |      |   table    |        |   ring   |
                   +----------+      +------------+        +----------+
                   |          |      |            |        |          |
                   +----------+      +------------+        +----------+
                   |          |      |            |        |          |
                   +----------+      +------------+        +----------+
                        \                   ^                   ^
                         \                  |                  /
                         get             update              put
                           \                |                /
                            V               |               /
                           +----------------+-------------------+
                           |       virtio host backend          |
                           +------------------------------------+
```

**Desriptor Table** 中存放的是一个一个的 **virtq_desc 元素**, **每个** virq_desc 元素占用 **16 个字节**(128位).

```
+-----------------------------------------------------------+
|                        addr/gpa [0: 63]                    |
+-------------------------+-----------------+---------------+
|         len [0: 31]      |  flags [0: 15]   |  next [0: 15]  |
+-------------------------+-----------------+---------------+
```

其中, addr 占用 64bit 存放了单个 IO 请求的 GPA 地址信息, 例如 addr 可能表示某个 DMA buffer 的起始地址. len 占用 32bit 表示 IO 请求的长度, flags 的取值有 3 种, VIRTQ_DESC_F_NEXT 表示这个 IO 请求和下一个 virtq_desc 描述的是连续的, IRTQ_DESC_F_WRITE 表示这段 buffer 是 write only 的, VIRTQ_DESC_F_INDIRECT 表示这段 buffer 里面放的内容是另外一组 buffer 的 virtq_desc (相当于重定向), next 是指向下一个 virtq_desc 的索引号 (前提是 VIRTQ_DESC_F_NEXT & flags).

Available Ring 是前端驱动用来告知后端那些 IO buffer 是的请求需要处理, 每个 Ring 中包含一个 virtq_avail 占用 8 个字节. 其中, flags 取值为 VIRTQ_AVAIL_F_NO_INTERRUPT 时表示前端驱动告诉后端: "当你消耗完一个 IO buffer 的时候, 不要立刻给我发中断"(防止中断过多影响效率). idx 表示下次前端驱动要放置 Descriptor Entry 的地方.

```
+--------------+-------------+--------------+---------------------+
| flags [0: 15] |  idx [0: 15] |  ring [0: 15]  |  used_event [0: 15]  |
+--------------+-------------+--------------+---------------------+
```

Used Ring 结构稍微不一样, flags 的值如果为 VIRTIO_F_EVENT_IDX 并且前后端协商 VIRTIO_F_EVENT_IDX feature 成功, 那么 Guest 会将 used ring index 放在 available ring 的末尾, 告诉后端说: "Hi 小老弟, 当你处理完这个请求的时候, 给我发个中断通知我一下", 同时 host 也会将 avail_event index 放到 used ring 的末尾, 告诉 guest 说: "Hi 老兄, 记得把这个 idx 的请求 kick 给我哈". VIRTIO_F_EVENT_IDX 对 virtio 通知 / 中断有一定的优化, 在某些场景下能够提升 IO 性能.

```cpp
/* The Guest publishes the used index for which it expects an interrupt
 * at the end of the avail ring. Host should ignore the avail->flags field. */
/* The Host publishes the avail index for which it expects a kick
 * at the end of the used ring. Guest should ignore the used->flags field. */

struct virtq_used {
#define VIRTQ_USED_F_NO_NOTIFY  1
        le16 flags;
        le16 idx;
        struct virtq_used_elem ring [ /* Queue Size */];
        le16 avail_event; /* Only if VIRTIO_F_EVENT_IDX */
};

/* le32 is used here for ids for padding reasons. */
struct virtq_used_elem {
        /* Index of start of used descriptor chain. */
        le32 id;
        /* Total length of the descriptor chain which was used (written to) */
        le32 len;
};
```

原理就到这里, 后面会以 virtio 网卡为例进行详细流程说明.

# 前后端通信机制 (irqfd 与 ioeventfd)

共享内存方式解决了传统设备 IO 过程中内存拷贝带来的性能损耗问题, 除此之外**前端驱动**和**后端驱动**的**通信**问题也是有可以改进的地方.

Virtio 前后端通信概括起来只有两个方向, 即 **GuestOS 通知 QEMU** 和 **QEMU 通知 GuestOS**. 当**前端驱动**准备好 **IO buffer** 之后, 需要**通知后端** (QEMU), 告诉后端: "我有一波 IO 请求已经准备好了, 你帮我处理一下". 前端通知出去后, 就可以等待 IO 结果了 (**操作系统可以进行一次调度**), 这时候 vCPU 可以去干点其他的事情. 后端收到消息后开始处理 IO 请求, 当 IO 请求处理完成之后, **后端**就通过**中断机制**通知 GuestOS: "你的 IO 给你处理好了, 你来取一下".

前后端通信机制如下图所示:

```
             +-------------+                +-------------+
             |             |                |             |
             |             |                |             |
             |   GuestOS   |                |     QEMU    |
             |             |                |             |
             |             |                |             |
             +---+---------+                +----+--------+
                 |     ^                         |    ^
                 |     |                         |    |
             +---|-----|-------------------------|----|---+
             |   |     |                irqfd    |    |   |
             |   |     +-------------------------+    |   |
             |   |  ioeventfd                         |   |
             |   +------------------------------------+   |
             |                   KVM                      |
             +--------------------------------------------+
```

**前端驱动通知后端**比较简单, **QEMU** 设置一段**特定的 MMIO 地址空间**, **前端驱动**访问这段 MMIO 触发 **VM-Exit**, 退出到 **KVM** 后利用 **ioeventfd** 机制**通知**到用户态的 **QEMU**, QEMU 主循环 (`main_loop poll`) 检测到 ioeventfd 事件后调用 **callback** 进行处理.

```cpp
前端驱动通知后端:
内核流程 mark 一下, PCI 设备驱动流程这个后面可以学习一下, 先扫描 PCI bus 发现是 virtio 设备再扫描 virtio-bus.
worker_thread --> process_one_work --> pciehp_power_thread --> pciehp_enable_slot -->
pciehp_configure_device --> pci_bus_add_devices --> pci_bus_add_device --> device_attach -->
__device_attach --> bus_for_each_drv --> __device_attach_driver --> driver_probe_device -->
pci_device_probe --> local_pci_probe --> virtio_pci_probe --> register_virtio_device -->
device_register --> device_add --> bus_probe_device --> device_initial_probe
--> __device_attach --> bus_for_each_drv --> __device_attach_driver -->
driver_probe_device --> virtio_dev_probe --> virtnet_probe (网卡设备驱动加载的入口)

static int virtnet_probe (struct virtio_device *vdev)
{
    ......
    virtio_device_ready (vdev);
}

/**
 * virtio_device_ready - enable vq use in probe function
 * @vdev: the device
 *
 * Driver must call this to use vqs in the probe function.
 *
 * Note: vqs are enabled automatically after probe returns.
 */
static inline
void virtio_device_ready (struct virtio_device *dev)
{
        unsigned status = dev->config->get_status (dev);

        BUG_ON (status & VIRTIO_CONFIG_S_DRIVER_OK);
        dev->config->set_status (dev, status | VIRTIO_CONFIG_S_DRIVER_OK);
}

# QEMU/KVM 后端的处理流程如下:
# 前端驱动写 Status 位, val & VIRTIO_CONFIG_S_DRIVER_OK, 这时候前端驱动已经 ready
virtio_pci_config_write  --> virtio_ioport_write --> virtio_pci_start_ioeventfd
--> virtio_bus_set_host_notifier --> virtio_bus_start_ioeventfd --> virtio_device_start_ioeventfd_impl
--> virtio_bus_set_host_notifier
    --> virtio_pci_ioeventfd_assign
        --> memory_region_add_eventfd
            --> memory_region_transaction_commit
              --> address_space_update_ioeventfds
                --> address_space_add_del_ioeventfds
                  --> kvm_io_ioeventfd_add/vhost_eventfd_add
                    --> kvm_set_ioeventfd_pio
                      --> kvm_vm_ioctl (kvm_state, KVM_IOEVENTFD, &kick)
```

其实, 这就是 QEMU 的 Fast MMIO 实现机制. 我们可以看到, QEMU 会为每个设备 MMIO 对应的 MemoryRegion 注册一个 ioeventfd. 最后调用了一个 KVM_IOEVENTFD ioctl 到 KVM 内核里面, 而在 KVM 内核中会将 MMIO 对应的 (gpa, len, eventfd) 信息会注册到 KVM_FAST_MMIO_BUS 上. 这样当 Guest 访问 MMIO 地址范围退出后 (触发 EPT Misconfig), KVM 会查询一下访问的 GPA 是否落在某段 MMIO 地址空间 range 内部, 如果是的话就直接写 eventfd 告知 QEMU, QEMU 就会从 coalesced mmio ring page 中取 MMIO 请求 (注: pio page 和 mmio page 是 QEMU 和 KVM 内核之间的共享内存页, 已经提前 mmap 好了).

```cpp
#kvm 内核代码 virt/kvm/eventfd.c 中
kvm_vm_ioctl (KVM_IOEVENTFD)
  --> kvm_ioeventfd
    --> kvm_assign_ioeventfd
      --> kvm_assign_ioeventfd_idx

# MMIO 处理流程中 (handle_ept_misconfig) 最后会调用到 ioeventfd_write 通知 QEMU.
/* MMIO/PIO writes trigger an event if the addr/val match */
static int
ioeventfd_write (struct kvm_vcpu *vcpu, struct kvm_io_device *this, gpa_t addr,
                int len, const void *val)
{
        struct _ioeventfd *p = to_ioeventfd (this);

        if (! ioeventfd_in_range (p, addr, len, val))
                return -EOPNOTSUPP;

        eventfd_signal (p->eventfd, 1);
        return 0;
}
```

不了解 MMIO 是如何模拟的童鞋, 可以结合本站的文章 MMIO 模拟实现分析去了解一下, 如果还是不懂的可以在文章下面评论.

后端通知前端, 是通过中断的方式, QEMU/KVM 中有一套完整的中断模拟实现框架,

如果对 QEMU/KVM 中断模拟不熟悉的童鞋, 建议阅读一下这篇文章: QEMU 学习笔记 - 中断. 对于 virtio-pci 设备, 可以通过 Cap 呈现 MSIx 给虚拟机, 这样在前端驱动加载的时候就会尝试去使能 MSIx 中断, 后端在这个时候建立起 MSIx 通道.

前端驱动加载 (probe) 的过程中, 会去初始化 virtqueue, 这个时候会去申请 MSIx 中断并注册中断处理函数:

```cpp
virtnet_probe
  --> init_vqs
    --> virtnet_find_vqs
      --> vi->vdev->config->find_vqs [vp_modern_find_vqs]
        --> vp_find_vqs
          --> vp_find_vqs_msix // 为每 virtqueue 申请一个 MSIx 中断, 通常收发各一个队列
            --> vp_request_msix_vectors // 主要的 MSIx 中断申请逻辑都在这个函数里面
              --> pci_alloc_irq_vectors_affinity // 申请 MSIx 中断描述符 (__pci_enable_msix_range)
                --> request_irq  // 注册中断处理函数

           //virtio-net 网卡至少申请了 3 个 MSIx 中断:
                // 一个是 configuration change 中断 (配置空间发生变化后, QEMU 通知前端)
                // 发送队列 1 个 MSIx 中断, 接收队列 1MSIx 中断
```

在 QEMU/KVM 这一侧, 开始模拟 MSIx 中断, 具体流程大致如下:

```cpp
virtio_pci_config_write
  --> virtio_ioport_write
    --> virtio_set_status
      --> virtio_net_vhost_status
        --> vhost_net_start
          --> virtio_pci_set_guest_notifiers
            --> kvm_virtio_pci_vector_use
              |--> kvm_irqchip_add_msi_route // 更新中断路由表
              |--> kvm_virtio_pci_irqfd_use  // 使能 MSI 中断
                 --> kvm_irqchip_add_irqfd_notifier_gsi
                   --> kvm_irqchip_assign_irqfd

# 申请 MSIx 中断的时候, 会为 MSIx 分配一个 gsi, 并为这个 gsi 绑定一个 irqfd, 然后调用 ioctl KVM_IRQFD 注册到内核中.
static int kvm_irqchip_assign_irqfd (KVMState *s, int fd, int rfd, int virq,
                                    bool assign)
{
    struct kvm_irqfd irqfd = {
        .fd = fd,
        .gsi = virq,
        .flags = assign ? 0 : KVM_IRQFD_FLAG_DEASSIGN,
    };

    if (rfd != -1) {
        irqfd.flags |= KVM_IRQFD_FLAG_RESAMPLE;
        irqfd.resamplefd = rfd;
    }

    if (! kvm_irqfds_enabled ()) {
        return -ENOSYS;
    }

    return kvm_vm_ioctl (s, KVM_IRQFD, &irqfd);
}

# KVM 内核代码 virt/kvm/eventfd.c
kvm_vm_ioctl (s, KVM_IRQFD, &irqfd)
  --> kvm_irqfd_assign
    --> vfs_poll (f.file, &irqfd->pt) // 在内核中 poll 这个 irqfd
```

从上面的流程可以看出, QEMU/KVM 使用 irqfd 机制来模拟 MSIx 中断, 即设备申请 MSIx 中断的时候会为 MSIx 分配一个 gsi (这个时候会刷新 irq routing table), 并为这个 gsi 绑定一个 irqfd, 最后在内核中去 poll 这个 irqfd. 当 QEMU 处理完 IO 之后, 就写 MSIx 对应的 irqfd, 给前端注入一个 MSIx 中断, 告知前端我已经处理好 IO 了你可以来取结果了.

例如, virtio-scsi 从前端取出 IO 请求后会取做 DMA 操作 (DMA 是异步的, QEMU 协程中负责处理). 当 DMA 完成后 QEMU 需要告知前端 IO 请求已完成 (Complete), 那么怎么去投递这个 MSIx 中断呢? 答案是调用 virtio_notify_irqfd 注入一个 MSIx 中断.

```
Copy
#0  0x00005604798d569b in virtio_notify_irqfd (vdev=0x56047d12d670, vq=0x7fab10006110) at  hw/virtio/virtio.c: 1684
#1  0x00005604798adea4 in virtio_scsi_complete_req (req=0x56047d09fa70) at  hw/scsi/virtio-scsi.c: 76
#2  0x00005604798aecfb in virtio_scsi_complete_cmd_req (req=0x56047d09fa70) at  hw/scsi/virtio-scsi.c: 468
#3  0x00005604798aee9d in virtio_scsi_command_complete (r=0x56047ccb0be0, status=0, resid=0) at  hw/scsi/virtio-scsi.c: 495
#4  0x0000560479b397cf in scsi_req_complete (req=0x56047ccb0be0, status=0) at hw/scsi/scsi-bus.c: 1404
#5  0x0000560479b2b503 in scsi_dma_complete_noio (r=0x56047ccb0be0, ret=0) at hw/scsi/scsi-disk.c: 279
#6  0x0000560479b2b610 in scsi_dma_complete (opaque=0x56047ccb0be0, ret=0) at hw/scsi/scsi-disk.c: 300
#7  0x00005604799b89e3 in dma_complete (dbs=0x56047c6e9ab0, ret=0) at dma-helpers.c: 118
#8  0x00005604799b8a90 in dma_blk_cb (opaque=0x56047c6e9ab0, ret=0) at dma-helpers.c: 136
#9  0x0000560479cf5220 in blk_aio_complete (acb=0x56047cd77d40) at block/block-backend.c: 1327
#10 0x0000560479cf5470 in blk_aio_read_entry (opaque=0x56047cd77d40) at block/block-backend.c: 1387
#11 0x0000560479df49c4 in coroutine_trampoline (i0=2095821104, i1=22020) at util/coroutine-ucontext.c: 115
#12 0x00007fab214d82c0 in __start_context () at /usr/lib64/libc.so.6
```

在 virtio_notify_irqfd 函数中, 会去写 irqfd, 给内核发送一个信号.

```cpp
void virtio_notify_irqfd (VirtIODevice *vdev, VirtQueue *vq)
{
    ...
     /*
     * virtio spec 1.0 says ISR bit 0 should be ignored with MSI, but
     * windows drivers included in virtio-win 1.8.0 (circa 2015) are
     * incorrectly polling this bit during crashdump and hibernation
     * in MSI mode, causing a hang if this bit is never updated.
     * Recent releases of Windows do not really shut down, but rather
     * log out and hibernate to make the next startup faster.  Hence,
     * this manifested as a more serious hang during shutdown with
     *
     * Next driver release from 2016 fixed this problem, so working around it
     * is not a must, but it's easy to do so let's do it here.
     *
     * Note: it's safe to update ISR from any thread as it was switched
     * to an atomic operation.
     */
    virtio_set_isr (vq->vdev, 0x1);
    event_notifier_set (&vq->guest_notifier);   // 写 vq->guest_notifier, 即 irqfd
}
```

QEMU 写了这个 irqfd 后, KVM 内核模块中的 irqfd poll 就收到一个 POLL_IN 事件, 然后将 MSIx 中断自动投递给对应的 LAPIC. 大致流程是: `POLL_IN -> kvm_arch_set_irq_inatomic -> kvm_set_msi_irq, kvm_irq_delivery_to_apic_fast`

```cpp
static int
irqfd_wakeup (wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
        if (flags & EPOLLIN) {
                idx = srcu_read_lock (&kvm->irq_srcu);
                do {
                        seq = read_seqcount_begin (&irqfd->irq_entry_sc);
                        irq = irqfd->irq_entry;
                } while (read_seqcount_retry (&irqfd->irq_entry_sc, seq));
                /* An event has been signaled, inject an interrupt */
                if (kvm_arch_set_irq_inatomic (&irq, kvm,
                                             KVM_USERSPACE_IRQ_SOURCE_ID, 1,
                                             false) == -EWOULDBLOCK)
                        schedule_work (&irqfd->inject);
                srcu_read_unlock (&kvm->irq_srcu, idx);
        }
```

这里还有一点没有想明白, 结合代码和调试来看, virtio-blk/virtio-scsi 的 msi 中断走 irqfd 机制, 但是 virtio-net (不开启 vhost 的情况下) 不走 irqfd, 而是直接调用 virtio_notify/virtio_pci_notify, 最后通过 KVM 的 ioctl 投递中断? 从代码路径上来看, 后者明显路径更长, 谁知道原因告诉我一下! ! ! . https://patchwork.kernel.org/patch/9531577/

```
Copy
Once in virtio_notify_irqfd, once in virtio_queue_guest_notifier_read.

Unfortunately, for virtio-blk + MSI + KVM + old Windows drivers we need the one in virtio_notify_irqfd.
For virtio-net + vhost + INTx we need the one in virtio_queue_guest_notifier_read.
这显然路径更长啊.
```

Ok, 到这里 virtio 前后端通信机制已经明了, 最后一个小节我们以 virtio-net 为例, 梳理一下 virtio 中的部分核心代码流程.

# virtio 核心代码分析, 以 virtio-net 为例

这里我们已 virtio-net 网卡为例, 在没有使用 vhost 的情况下 (网卡后端收发包都走 QEMU 处理), 后端收发包走 vhost 的情况下有些不同, 后面单独分析.

## 前后端握手流程

QEMU 模拟 PCI 设备对 GuestOS 进行呈现, **设备驱动**加载的时候尝试去**初始化设备**.

```cpp
# 先在 PCI 总线上调用 probe 设备, 调用了 virtio_pci_probe, 然后再 virtio-bus 上调用 virtio_dev_probe
# virtio_dev_probe 最后调用到 virtnet_probe
pci_device_probe --> local_pci_probe --> virtio_pci_probe --> register_virtio_device -->
device_register --> device_add --> bus_probe_device --> device_initial_probe
--> __device_attach --> bus_for_each_drv --> __device_attach_driver --> driver_probe_device -->
virtio_dev_probe --> virtnet_probe

// 在 virtio_pci_probe 里先尝试以 virtio modern 方式读取设备配置数据结构, 如果失败则尝试 virio legacy 方式.
// 对于 virtio legacy, 我们前面提到了 virtio legacy 协议规定设备的配置数据结构放在 PCI BAR0 里面.
/* the PCI probing function */
int virtio_pci_legacy_probe (struct virtio_pci_device *vp_dev)
{
        rc = pci_request_region (pci_dev, 0, "virtio-pci-legacy");  // 将设备的 BAR0 映射到物理地址空间
        vp_dev->ioaddr = pci_iomap (pci_dev, 0, 0);   // 获得 BAR0 的内核地址
}

# 对于 virtio modern, 通过 capability 方式报告配置数据结构的位置, 配置数据结构有 5 种类型.
int virtio_pci_modern_probe (struct virtio_pci_device *vp_dev)
{
        /* check for a common config: if not, use legacy mode (bar 0). */
        common = virtio_pci_find_capability (pci_dev, VIRTIO_PCI_CAP_COMMON_CFG,
                                            IORESOURCE_IO | IORESOURCE_MEM,
                                            &vp_dev->modern_bars);

        /* If common is there, these should be too... */
        isr = virtio_pci_find_capability (pci_dev, VIRTIO_PCI_CAP_ISR_CFG,
                                         IORESOURCE_IO | IORESOURCE_MEM,
                                         &vp_dev->modern_bars);
        notify = virtio_pci_find_capability (pci_dev, VIRTIO_PCI_CAP_NOTIFY_CFG,
                                            IORESOURCE_IO | IORESOURCE_MEM,
                                            &vp_dev->modern_bars);

        /* Device capability is only mandatory for devices that have
        * device-specific configuration.
        */
        device = virtio_pci_find_capability (pci_dev, VIRTIO_PCI_CAP_DEVICE_CFG,
                                            IORESOURCE_IO | IORESOURCE_MEM,
                                            &vp_dev->modern_bars);

        err = pci_request_selected_regions (pci_dev, vp_dev->modern_bars,
                                            "virtio-pci-modern");
                                        sizeof (struct virtio_pci_common_cfg), 4,
                                        0, sizeof (struct virtio_pci_common_cfg),
                                        NULL);
        // 将配 virtio 置结构所在的 BAR 空间 MAP 到内核地址空间里
        vp_dev->common = map_capability (pci_dev, common,
                                        sizeof (struct virtio_pci_common_cfg), 4,
                                        0, sizeof (struct virtio_pci_common_cfg),
                                        NULL);
        ......
}

# 接着来到 virtio_dev_probe 里面看下:
static int virtio_dev_probe (struct device *_d)
{
        /* We have a driver! */
        virtio_add_status (dev, VIRTIO_CONFIG_S_DRIVER);     // 更新 status bit, 这里要写配置数据结构

        /* Figure out what features the device supports. */
        device_features = dev->config->get_features (dev);   // 查询后端支持哪些 feature bits

        //feature set 协商, 取交集
        err = virtio_finalize_features (dev);

        // 调用特定 virtio 设备的驱动程序 probe, 例如: virtnet_probe, virtblk_probe
        err = drv->probe (dev);
}
```

再看下 virtnet_probe 里面的一些关键的流程, 这里包含了 virtio-net 网卡前端初始化的主要逻辑.

```cpp
Copy
static int virtnet_probe (struct virtio_device *vdev)
{
       //check 后端是否支持多队列, 并按情况创建队列
       /* Allocate ourselves a network device with room for our info */
        dev = alloc_etherdev_mq (sizeof (struct virtnet_info), max_queue_pairs);

        // 定义一个网络设备并配置一些属性, 例如 MAC 地址
        dev->ethtool_ops = &virtnet_ethtool_ops;
           SET_NETDEV_DEV (dev, &vdev->dev);

        // 初始化 virtqueue
        err = init_vqs (vi);

        // 注册一个网络设备
        err = register_netdev (dev);

        // 写状态位 DRIVER_OK, 告诉后端, 前端已经 ready
        virtio_device_ready (vdev);

        // 将网卡 up 起来
        netif_carrier_on (dev);
}
```

其中关键的流程是 init_vqs, 在 vp_find_vqs_msix 流程中会尝试去申请 MSIx 中断, 这里前面已经有分析过了. 其中, "configuration changed" 中断服务程序 vp_config_changed, virtqueue 队列的中断服务程序是 vp_vring_interrupt.

```cpp
Copy
init_vqs --> virtnet_find_vqs --> vi->vdev->config->find_vqs --> vp_modern_find_vqs
--> vp_find_vqs --> vp_find_vqs_msix

static int vp_find_vqs_msix (struct virtio_device *vdev, unsigned nvqs,
        struct virtqueue *vqs [], vq_callback_t *callbacks [],
        const char * const names [], bool per_vq_vectors,
        const bool *ctx,
        struct irq_affinity *desc)
{
        /* 为 configuration change 申请 MSIx 中断 */
    err = vp_request_msix_vectors (vdev, nvectors, per_vq_vectors,
                  per_vq_vectors ? desc : NULL);
        for (i = 0; i < nvqs; ++i) {
         // 创建队列 --> vring_create_virtqueue --> vring_create_virtqueue_split --> vring_alloc_queue
             vqs [i] = vp_setup_vq (vdev, queue_idx++, callbacks [i], names [i],
                                ctx ? ctx [i] : false,
                                msix_vec);
        // 每个队列申请一个 MSIx 中断
                err = request_irq (pci_irq_vector (vp_dev->pci_dev, msix_vec),
                                  vring_interrupt, 0,
                                  vp_dev->msix_names [msix_vec],
                                  vqs [i]);
    }
```

vp_setup_vq 流程再往下走就开始分配共享内存页, 至此建立起共享内存通信通道. 值得注意的是一路传下来的 callbacks 参数其实传入了发送队列和接收队列的回调处理函数, 好家伙, 从 virtnet_find_vqs 一路传递到了__vring_new_virtqueue 中最终赋值给了 vq->vq.callback.

```cpp
Copy
static struct virtqueue *vring_create_virtqueue_split (
        unsigned int index,
        unsigned int num,
        unsigned int vring_align,
        struct virtio_device *vdev,
        bool weak_barriers,
        bool may_reduce_num,
        bool context,
        bool (*notify)(struct virtqueue *),
        void (*callback)(struct virtqueue *),
        const char *name)
{
       /* TODO: allocate each queue chunk individually */
        for (; num && vring_size (num, vring_align) > PAGE_SIZE; num /= 2) {
        // 申请物理页, 地址赋值给 queue
                queue = vring_alloc_queue (vdev, vring_size (num, vring_align),
                                          &dma_addr,
                                          GFP_KERNEL|__GFP_NOWARN|__GFP_ZERO);
        }


        queue_size_in_bytes = vring_size (num, vring_align);
        vring_init (&vring, num, queue, vring_align); // 确定 descriptor table, available ring, used ring 的位置.
}
```

我们看下如果 virtqueue 队列如果收到 MSIx 中断消息后, 会调用哪个 hook 来处理?

```cpp
Copy
irqreturn_t vring_interrupt (int irq, void *_vq)
{
        struct vring_virtqueue *vq = to_vvq (_vq);

        if (! more_used (vq)) {
                pr_debug ("virtqueue interrupt with no work for % p\n", vq);
                return IRQ_NONE;
        }

        if (unlikely (vq->broken))
                return IRQ_HANDLED;

        pr_debug ("virtqueue callback for % p (% p)\n", vq, vq->vq.callback);
        if (vq->vq.callback)
                vq->vq.callback (&vq->vq);

        return IRQ_HANDLED;
}
EXPORT_SYMBOL_GPL (vring_interrupt);
```

不难想到中断服务程序里面会调用队列上的 callback. 我们再回过头来看下 virtnet_find_vqs, 原来接受队列的回调函数是 skb_recv_done, 发送队列的回调函数是 skb_xmit_done.

```cpp
Copy
static int virtnet_find_vqs (struct virtnet_info *vi)
{
       /* Allocate/initialize parameters for send/receive virtqueues */
        for (i = 0; i < vi->max_queue_pairs; i++) {
        callbacks [rxq2vq (i)] = skb_recv_done;
        callbacks [txq2vq (i)] = skb_xmit_done;
    }
}
```

## virtio-net 网卡收发在 virtqueue 上的实现
这里以 virtio-net 为例 (非 vhost-net 模式) 来分析一下网卡收发报文在 virtio 协议上的具体实现. virtio-net 模式下网卡收发包的流程为:

* 收包: Hardware => Host Kernel => Qemu => Guest
* 发包: Guest => Host Kernel => Qemu => Host Kernel => Hardware

### virtio-net 网卡发包

前面我们看到 virtio-net 设备初始化的时候会创建一个 net_device 设备: virtnet_probe -> alloc_etherdev_mq 注册了 netdev_ops = &virtnet_netdev, 这里 virtnet_netdev 是网卡驱动的回调函数集合 (收发包和参数设置).

```cpp
static const struct net_device_ops netdev_ops = {
        .ndo_open               = rio_open,
        .ndo_start_xmit = start_xmit,
        .ndo_stop               = rio_close,
        .ndo_get_stats          = get_stats,
        .ndo_validate_addr      = eth_validate_addr,
        .ndo_set_mac_address    = eth_mac_addr,
        .ndo_set_rx_mode        = set_multicast,
        .ndo_do_ioctl           = rio_ioctl,
        .ndo_tx_timeout         = rio_tx_timeout,
};
```

网卡发包的时候调用 ndo_start_xmit, 将 TCP/IP 上层协议栈扔下来的数据发送出去. 对应到 virtio 网卡的回调函数就是 start_xmit, 从代码看就是将 skb 发送到 virtqueue 中, 然后调用 virtqueue_kick 通知 qemu 后端将数据包发送出去.

Guest 内核里面的 virtio-net 驱动发包:

```
Copy
内核驱动 virtio_net.c
start_xmit
    // 将 skb 放到 virtqueue 队列中
    -> xmit_skb -> sg_init_table, virtqueue_add_outbuf -> virtqueue_add
    //kick 通知 qemu 后端去取
    virtqueue_kick_prepare && virtqueue_notify
    //kick 次数加 1
    sq->stats.kicks++
```

Guest Kick 后端从 KVM 中 VMExit 出来退出到 Qemu 用户态 (走的是 ioeventfd) 由 Qemu 去将数据发送出去. 大致调用的流程是: virtio_queue_host_notifier_read -> virtio_net_handle_tx_bh -> virtio_net_flush_tx -> virtqueue_pop 拿到发包 (skb) -> qemu_sendv_packet_async

```
Copy
Qemu 代码 virtio-net 相关代码:
virtio_queue_host_notifier_read -> virtio_queue_notify_vq
    -> vq->handle_output -> virtio_net_handle_tx_bh 队列注册的时候, 回注册回调函数
        -> qemu_bh_schedule -> virtio_net_tx_bh
            -> virtio_net_flush_tx
            -> virtqueue_pop
        -> qemu_sendv_packet_async // 报文放到发送队列上, 写 tap 设备的 fd 去发包
            -> tap_receive_iov -> tap_write_packet

// 最后调用 tap_write_packet 把数据包发给 tap 设备投递出去
```

### virtio-net 网卡收包

网卡收包的时候, tap 设备先收到报文, 对应的 virtio-net 网卡 tap 设备 fd 变为可读, Qemu 主循环收到 POLL_IN 事件调用回调函数收包.

```
Copy
tap_send -> qemu_send_packet_async -> qemu_send_packet_async_with_flags
    -> qemu_net_queue_send
        -> qemu_net_queue_deliver
        -> qemu_deliver_packet_iov
            -> nc_sendv_compat
            -> virtio_net_receive
                -> virtio_net_receive_rcu
```

virtio-net 网卡收报最终调用了 virtio_net_receive_rcu, 和发包类似都是调用 virtqueue_pop 从前端获取 virtqueue element, 将报文数据填充到 vring 中然后 virtio_notify 注入中断通知前端驱动取结果.

这里不得不吐槽一下, 为啥收包函数取名要叫 tap_send.


# reference

https://kernelgo.org/virtio-overview.html

1. [virtio spec v1.1](https://docs.oasis-open.org/virtio/virtio/v1.1/csprd01/virtio-v1.1-csprd01.html)
2. [Towards a De-Facto Standard For Virtual](https://ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf)
3. https://github.com/qemu/qemu/blob/master/hw/net/virtio-net.c
4. https://github.com/torvalds/linux/blob/master/drivers/net/virtio_net.c