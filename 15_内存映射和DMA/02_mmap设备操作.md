# 15.2. mmap设备操作
内存映射是现代 Unix 系统最有趣的特性之一。就驱动程序而言，实现内存映射可以为用户程序提供对设备内存的直接访问。

通过查看 X 窗口系统服务器的部分虚拟内存区域，可以看到 mmap 使用的一个典型示例：
```plaintext
cat /proc/731/maps
000a0000 - 000c0000 rwxs 000a0000 03:01 282652      /dev/mem
000f0000 - 00100000 r-xs 000f0000 03:01 282652      /dev/mem
00400000 - 005c0000 r-xp 00000000 03:01 1366927     /usr/X11R6/bin/Xorg
006bf000 - 006f7000 rw-p 001bf000 03:01 1366927     /usr/X11R6/bin/Xorg
2a95828000 - 2a958a8000 rw-s fcc00000 03:01 282652  /dev/mem
2a958a8000 - 2a9d8a8000 rw-s e8000000 03:01 282652  /dev/mem
...
```
X 服务器的虚拟内存区域（VMA）完整列表很长，但这里大多数条目并非我们所关注的。不过，我们确实看到了 `/dev/mem` 的四个不同映射，这让我们对 X 服务器如何与显卡协同工作有了一些了解。第一个映射地址是 `a0000`，这是 640KB ISA 空洞中视频 RAM 的标准位置。再往下，我们看到一个大的映射地址为 `e8000000`，这个地址高于系统中最高的 RAM 地址，这是对适配器上视频内存的直接映射。

这些区域在 `/proc/iomem` 中也能看到：
```plaintext
000a0000 - 000bffff : Video RAM area
000c0000 - 000ccfff : Video ROM
000d1000 - 000d1fff : Adapter ROM
000f0000 - 000fffff : System ROM
d7f00000 - f7efffff : PCI Bus #01
  e8000000 - efffffff : 0000:01:00.0
fc700000 - fccfffff : PCI Bus #01
  fcc00000 - fcc0ffff : 0000:01:00.0
```
映射设备意味着将用户空间的一段地址范围与设备内存关联起来。每当程序在指定的地址范围内进行读写操作时，实际上是在访问设备。在 X 服务器的例子中，使用 mmap 可以快速方便地访问显卡内存。对于像这样对性能要求很高的应用程序来说，直接访问会带来很大的不同。

正如你可能猜到的，并非每个设备都适合使用 mmap 抽象；例如，对于串行端口和其他面向流的设备来说，使用 mmap 就没有意义。mmap 的另一个限制是映射是以页面大小（`PAGE_SIZE`）为粒度的。内核只能在页表级别管理虚拟地址；因此，映射区域的大小必须是 `PAGE_SIZE` 的倍数，并且物理内存的起始地址也必须是 `PAGE_SIZE` 的倍数。如果区域大小不是页面大小的倍数，内核会将区域稍微扩大以满足粒度要求。

这些限制对驱动程序来说并不是很大的约束，因为访问设备的程序本身就依赖于设备。由于程序必须了解设备的工作方式，程序员不会因处理页面对齐等细节而感到过分困扰。当在一些非 x86 平台上使用 ISA 设备时，会存在一个更大的约束，因为这些平台对 ISA 的硬件视图可能不是连续的。例如，一些 Alpha 计算机将 ISA 内存视为分散的 8 位、16 位或 32 位数据项的集合，没有直接映射关系。在这种情况下，根本无法使用 mmap。无法将 ISA 地址直接映射到 Alpha 地址是由于这两个系统不兼容的数据传输规范造成的。早期的 Alpha 处理器只能进行 32 位和 64 位的内存访问，而 ISA 只能进行 8 位和 16 位的传输，并且无法将一种协议透明地映射到另一种协议上。

在可行的情况下，使用 mmap 有明显的优势。例如，我们已经看到了 X 服务器，它需要频繁地在视频内存中传输大量数据；将图形显示映射到用户空间与使用 `lseek/write` 实现相比，显著提高了吞吐量。另一个典型的例子是控制 PCI 设备的程序。大多数 PCI 外设将其控制寄存器映射到一个内存地址，高性能应用程序可能更倾向于直接访问这些寄存器，而不是反复调用 `ioctl` 来完成工作。

`mmap` 方法是 `file_operations` 结构体的一部分，当调用 `mmap` 系统调用时会被调用。使用 `mmap` 时，内核在调用实际的方法之前会做大量工作，因此该方法的原型与系统调用的原型有很大不同。这与 `ioctl` 和 `poll` 等调用不同，在调用这些方法之前，内核不会做太多工作。

系统调用的声明如下（如 `mmap(2)` 手册页所述）：
```c
mmap (caddr_t addr, size_t len, int prot, int flags, int fd, off_t offset)
```
另一方面，文件操作的声明为：
```c
int (*mmap) (struct file *filp, struct vm_area_struct *vma);
```
该方法中的 `filp` 参数与第 3 章中介绍的相同，而 `vma` 包含用于访问设备的虚拟地址范围的信息。因此，大部分工作已由内核完成；为了实现 `mmap`，驱动程序只需为该地址范围构建合适的页表，如果需要，用一组新的操作替换 `vma->vm_ops`。

构建页表有两种方法：一种是使用 `remap_pfn_range` 函数一次性完成，另一种是通过 `nopage` VMA 方法逐页完成。每种方法都有其优缺点。我们先从简单的“一次性”方法开始，然后再介绍实际实现中需要考虑的复杂情况。

# 15.2.1. 使用 remap_pfn_range
构建新的页表以映射一段物理地址的工作由 `remap_pfn_range` 和 `io_remap_page_range` 处理，它们的原型如下：
```c
int remap_pfn_range(struct vm_area_struct *vma, 
                     unsigned long virt_addr, unsigned long pfn,
                     unsigned long size, pgprot_t prot);
int io_remap_page_range(struct vm_area_struct *vma, 
                        unsigned long virt_addr, unsigned long phys_addr,
                        unsigned long size, pgprot_t prot);
```
该函数返回的值通常是 0 或一个负的错误码。让我们来看看这些函数参数的确切含义：
- **vma**：要将页面范围映射到的虚拟内存区域。
- **virt_addr**：重映射开始的用户虚拟地址。该函数为从 `virt_addr` 到 `virt_addr + size` 的虚拟地址范围构建页表。
- **pfn**：与虚拟地址应映射到的物理地址对应的页帧号。页帧号就是将物理地址右移 `PAGE_SHIFT` 位。在大多数情况下，VMA 结构体的 `vm_pgoff` 字段正好包含所需的值。该函数影响从 `(pfn << PAGE_SHIFT)` 到 `(pfn << PAGE_SHIFT) + size` 的物理地址。
- **size**：要重映射区域的大小，以字节为单位。
- **prot**：对新 VMA 请求的“保护”。驱动程序可以（并且应该）使用 `vma->vm_page_prot` 中的值。

`remap_pfn_range` 的参数相当直观，当调用 `mmap` 方法时，其中大部分参数已经在 VMA 中提供给你了。不过，你可能会疑惑为什么有两个函数。第一个函数 `remap_pfn_range` 适用于 `pfn` 指向实际系统 RAM 的情况，而 `io_remap_page_range` 应在 `phys_addr` 指向 I/O 内存时使用。实际上，除了 SPARC 架构外，这两个函数在所有架构上都是相同的，并且在大多数情况下都会使用 `remap_pfn_range`。然而，为了编写可移植的驱动程序，你应该根据具体情况选择合适的 `remap_pfn_range` 变体。

另一个复杂的问题与缓存有关：通常，对设备内存的引用不应被处理器缓存。系统 BIOS 通常会正确设置这些，但也可以通过保护字段禁用特定 VMA 的缓存。不幸的是，在这个层面禁用缓存高度依赖于处理器。好奇的读者可以查看 `drivers/char/mem.c` 中的 `pgprot_noncached` 函数，了解其中的细节。这里我们不再进一步讨论这个话题。

# 15.2.2. 简单实现
如果你的驱动程序需要将设备内存简单线性地映射到用户地址空间，`remap_pfn_range` 几乎可以完成所有工作。以下代码源自 `drivers/char/mem.c`，展示了在一个名为 `simple`（简单实现，不太热衷于映射页面）的典型模块中如何执行此任务：
```c
static int simple_remap_mmap(struct file *filp, struct vm_area_struct *vma)
{
    if (remap_pfn_range(vma, vma->vm_start, vma->vm_pgoff,
                vma->vm_end - vma->vm_start,
                vma->vm_page_prot))
        return -EAGAIN;

    vma->vm_ops = &simple_remap_vm_ops;
    simple_vma_open(vma);
    return 0;
}
```
如你所见，重映射内存只是调用 `remap_pfn_range` 来创建必要的页表的问题。

# 15.2.3. 添加 VMA 操作
正如我们所见，`vm_area_struct` 结构体包含一组可以应用于 VMA 的操作。现在我们来看看如何以简单的方式提供这些操作。具体来说，我们为 VMA 提供 `open` 和 `close` 操作。每当进程打开或关闭 VMA 时，这些操作就会被调用；特别是，每当进程分叉并创建对 VMA 的新引用时，`open` 方法就会被调用。`open` 和 `close` VMA 方法是在内核执行的处理之外被调用的，因此它们不需要重新实现内核已经完成的任何工作。它们的存在是为了让驱动程序能够进行任何额外需要的处理。

事实证明，像 `simple` 这样的简单驱动程序不需要进行任何特别的额外处理。所以我们创建了 `open` 和 `close` 方法，它们会向系统日志打印一条消息，告知调用了这些方法。虽然不是特别有用，但它展示了如何提供这些方法，以及何时调用它们。

为此，我们用调用 `printk` 的操作覆盖默认的 `vma->vm_ops`：
```c
void simple_vma_open(struct vm_area_struct *vma)
{
    printk(KERN_NOTICE "Simple VMA open, virt %lx, phys %lx\n",
            vma->vm_start, vma->vm_pgoff << PAGE_SHIFT);
}

void simple_vma_close(struct vm_area_struct *vma)
{
    printk(KERN_NOTICE "Simple VMA close.\n");
}

static struct vm_operations_struct simple_remap_vm_ops = {
    .open =  simple_vma_open,
    .close = simple_vma_close,
};
```
为了使这些操作对特定映射生效，需要将指向 `simple_remap_vm_ops` 的指针存储在相关 VMA 的 `vm_ops` 字段中。这通常在 `mmap` 方法中完成。如果你回头看 `simple_remap_mmap` 示例，会看到以下代码行：
```c
vma->vm_ops = &simple_remap_vm_ops;
simple_vma_open(vma);
```
注意对 `simple_vma_open` 的显式调用。由于在初始 `mmap` 时不会调用 `open` 方法，如果我们希望它运行，就必须显式调用它。

# 15.2.4 使用 nopage 映射内存
尽管 `remap_pfn_range` 对许多（即便不是大多数）驱动的 `mmap` 实现来说都很有效，但有时需要更灵活一些。在这种情况下，可能需要使用 `nopage` VMA 方法来实现。

`nopage` 方法在一种场景下会很有用，即应用程序使用 `mremap` 系统调用来更改映射区域的边界地址时。实际上，当通过 `mremap` 更改已映射的 VMA 时，内核不会直接通知驱动。如果 VMA 大小缩小，内核可以在不告知驱动的情况下悄悄清除不需要的页面。相反，如果 VMA 大小增大，当需要为新页面建立映射时，驱动最终会通过 `nopage` 调用得到通知，所以无需单独通知。因此，如果要支持 `mremap` 系统调用，就必须实现 `nopage` 方法。下面展示一个简单设备的 `nopage` 方法的简单实现。

请记住，`nopage` 方法的原型如下：
```c
struct page *(*nopage)(struct vm_area_struct *vma, 
                       unsigned long address, int *type);
```
当用户进程尝试访问 VMA 中当前不在内存里的页面时，会调用关联的 `nopage` 函数。`address` 参数包含导致缺页的虚拟地址，该地址会向下舍入到页面起始位置。`nopage` 函数必须定位并返回指向用户所需页面的 `struct page` 指针。此函数还必须通过调用 `get_page` 宏来增加返回页面的使用计数：
```c
get_page(struct page *pageptr);
```
这一步对于确保映射页面的引用计数正确是必要的。内核为每个页面维护这个计数；当计数变为 0 时，内核知道该页面可以放入空闲列表。当 VMA 被取消映射时，内核会减少该区域中每个页面的使用计数。如果驱动在向区域中添加页面时不增加计数，使用计数会过早变为 0，从而影响系统的完整性。

`nopage` 方法还应将缺页类型存储在 `type` 参数指向的位置——但前提是该参数不为 `NULL`。在设备驱动中，`type` 的正确值始终是 `VM_FAULT_MINOR`。

如果使用 `nopage`，在调用 `mmap` 时通常没多少工作要做；我们的实现如下：
```c
static int simple_nopage_mmap(struct file *filp, struct vm_area_struct *vma)
{
    unsigned long offset = vma->vm_pgoff << PAGE_SHIFT;

    if (offset >= __pa(high_memory) || (filp->f_flags & O_SYNC))
        vma->vm_flags |= VM_IO;
    vma->vm_flags |= VM_RESERVED;

    vma->vm_ops = &simple_nopage_vm_ops;
    simple_vma_open(vma);
    return 0;
}
```
`mmap` 主要的工作是用我们自己的操作替换默认的（`NULL`）`vm_ops` 指针。然后 `nopage` 方法负责逐页进行“重映射”，并返回其 `struct page` 结构的地址。由于这里只是实现对物理内存的一个访问窗口，重映射步骤很简单：只需定位并返回指向所需地址的 `struct page` 指针。我们的 `nopage` 方法如下：
```c
struct page *simple_vma_nopage(struct vm_area_struct *vma,
                unsigned long address, int *type)
{
    struct page *pageptr;
    unsigned long offset = vma->vm_pgoff << PAGE_SHIFT;
    unsigned long physaddr = address - vma->vm_start + offset;
    unsigned long pageframe = physaddr >> PAGE_SHIFT;

    if (!pfn_valid(pageframe))
        return NOPAGE_SIGBUS;
    pageptr = pfn_to_page(pageframe);
    get_page(pageptr);
    if (type)
        *type = VM_FAULT_MINOR;
    return pageptr;
}
```
同样，由于这里只是映射主内存，`nopage` 函数只需为出错地址找到正确的 `struct page` 并增加其引用计数。因此，所需的操作顺序是计算所需的物理地址，然后将其右移 `PAGE_SHIFT` 位转换为页帧号。由于用户空间可以传入任意地址，必须确保得到的是有效的页帧；`pfn_valid` 函数可以完成这个检查。如果地址超出范围，返回 `NOPAGE_SIGBUS`，这会导致向调用进程发送一个总线信号。否则，`pfn_to_page` 获取必要的 `struct page` 指针；增加其引用计数（通过调用 `get_page`）并返回该指针。

`nopage` 方法通常返回一个指向 `struct page` 的指针。如果由于某些原因无法返回正常页面（例如，请求的地址超出了设备的内存区域），可以返回 `NOPAGE_SIGBUS` 来表示错误；上面的简单代码就是这样做的。`nopage` 也可以返回 `NOPAGE_OOM` 来表示因资源限制导致的失败。

注意，此实现适用于 ISA 内存区域，但不适用于 PCI 总线的内存区域。PCI 内存映射在系统最高内存地址之上，系统内存映射中没有这些地址的条目。由于没有 `struct page` 指针可返回，在这些情况下不能使用 `nopage`；必须使用 `remap_pfn_range`。

如果 `nopage` 方法为 `NULL`，处理页面错误的内核代码会将零页面映射到出错的虚拟地址。零页面是一个写时复制页面，读取时为 0，例如用于映射 BSS 段。任何引用零页面的进程看到的都是一个全零的页面。如果进程向该页面写入数据，最终会修改一个私有副本。因此，如果进程通过调用 `mremap` 扩展映射区域，而驱动没有实现 `nopage` 方法，进程最终得到的将是全零填充的内存，而不是段错误。

# 15.2.5 重映射特定的 I/O 区域
到目前为止，我们看到的所有示例都是 `/dev/mem` 的重新实现；它们将物理地址重映射到用户空间。然而，典型的驱动程序通常只想映射其外围设备对应的小地址范围，而不是整个内存。为了只将整个内存范围的一部分映射到用户空间，驱动程序只需处理偏移量即可。以下代码展示了如何将一个大小为 `simple_region_size` 字节、起始物理地址为 `simple_region_start`（该地址应按页对齐）的区域进行映射：
```c
unsigned long off = vma->vm_pgoff << PAGE_SHIFT;
unsigned long physical = simple_region_start + off;
unsigned long vsize = vma->vm_end - vma->vm_start;
unsigned long psize = simple_region_size - off;

if (vsize > psize)
    return -EINVAL; /* 映射范围过大 */
remap_pfn_range(vma, vma->vm_start, physical, vsize, vma->vm_page_prot);
```
除了计算偏移量，这段代码还增加了一个检查，当程序试图映射的内存超过目标设备 I/O 区域的可用内存时，会报告错误。在这段代码中，`psize` 是指定偏移量后剩余的物理 I/O 大小，`vsize` 是请求的虚拟内存大小；该函数拒绝映射超出允许内存范围的地址。

需要注意的是，用户进程始终可以使用 `mremap` 来扩展其映射范围，有可能会超出物理设备区域的末尾。如果驱动程序没有定义 `nopage` 方法，将不会收到关于此扩展的通知，额外的区域将被映射到零页面。作为驱动程序开发者，可能希望避免这种情况；将零页面映射到区域末尾本身并不是明确的错误行为，但程序员很可能并不希望出现这种情况。

防止映射扩展的最简单方法是实现一个简单的 `nopage` 方法，该方法始终向出错的进程发送总线信号。这样的方法如下所示：
```c
struct page *simple_nopage(struct vm_area_struct *vma,
                           unsigned long address, int *type);
{ return NOPAGE_SIGBUS; /* 发送 SIGBUS 信号 */}
```
正如我们所见，`nopage` 方法仅在进程引用已知 VMA 内但当前没有有效页表条目的地址时才会被调用。如果我们使用 `remap_pfn_range` 映射了整个设备区域，这里展示的 `nopage` 方法仅在引用该区域之外的地址时才会被调用。因此，它可以安全地返回 `NOPAGE_SIGBUS` 来表示错误。当然，更完善的 `nopage` 实现可以检查出错地址是否在设备区域内，如果是，则执行重映射操作。不过，再次强调，`nopage` 不适用于 PCI 内存区域，因此无法扩展 PCI 映射。

# 15.2.6 重映射 RAM
`remap_pfn_range` 有一个有趣的限制，即它只能访问保留页面和物理内存顶部以上的物理地址。在 Linux 中，物理地址页面在内存映射中被标记为“保留”，表示该页面不可用于内存管理。例如，在 PC 上，640KB 到 1MB 之间的范围被标记为保留，内核代码所在的页面也是如此。保留页面被锁定在内存中，并且是唯一可以安全映射到用户空间的页面；这个限制是系统稳定性的基本要求。

因此，`remap_pfn_range` 不允许重映射常规地址，包括通过调用 `get_free_page` 获得的地址。相反，它会映射零页面。除了进程看到的是私有、全零填充的页面而不是期望的重映射 RAM 之外，一切似乎都能正常工作。尽管如此，该函数可以完成大多数硬件驱动所需的工作，因为它可以重映射高位 PCI 缓冲区和 ISA 内存。

通过运行 `mapper` 程序可以看到 `remap_pfn_range` 的局限性，`mapper` 是 O'Reilly 的 FTP 站点提供的文件中 `misc-progs` 目录下的示例程序之一。`mapper` 是一个简单的工具，可用于快速测试 `mmap` 系统调用；它根据命令行选项映射文件的只读部分，并将映射区域输出到标准输出。例如，以下会话显示 `/dev/mem` 不会映射位于地址 64KB 的物理页面——相反，我们看到的是一个全零的页面（此示例中的主机是 PC，但在其他平台上结果相同）：
```plaintext
morgana.root# ./mapper /dev/mem 0x10000 0x1000 | od -Ax -t x1
mapped "/dev/mem" from 65536 to 69632
000000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
*
001000
```
`remap_pfn_range` 无法处理 RAM 这一问题表明，像 `scull` 这样基于内存的设备难以轻松实现 `mmap`，因为其设备内存是常规 RAM，而不是 I/O 内存。幸运的是，对于任何需要将 RAM 映射到用户空间的驱动程序，有一个相对简单的解决方法；可以使用我们之前介绍的 `nopage` 方法。

## 15.2.6.1 使用 nopage 方法重映射 RAM
将实际 RAM 映射到用户空间的方法是使用 `vm_ops->nopage` 逐页处理页面错误。第 8 章介绍的 `scullp` 模块中有一个示例实现。

`scullp` 是一个面向页面的字符设备。由于它面向页面，因此可以在其内存上实现 `mmap`。实现内存映射的代码使用了 15.1 节中介绍的一些概念。

在查看代码之前，让我们先看看影响 `scullp` 中 `mmap` 实现的设计选择：
- `scullp` 在设备被映射时不会释放设备内存。这是一种策略选择，而非必要条件，与 `scull` 等类似设备的行为不同，`scull` 在以写模式打开时会被截断为长度为 0。拒绝释放已映射的 `scullp` 设备允许一个进程覆盖另一个进程正在映射的区域，因此可以测试进程与设备内存之间的交互。为了避免释放已映射的设备，驱动程序必须记录活动映射的数量；设备结构中的 `vmas` 字段用于此目的。
- 只有当 `scullp` 的 `order` 参数（在模块加载时设置）为 0 时才执行内存映射。该参数控制 `__get_free_pages` 的调用方式（见 8.3 节）。零阶限制（即强制每次只分配一个页面，而不是成组分配）是由 `scullp` 使用的分配函数 `__get_free_pages` 的内部机制决定的。为了最大化分配性能，Linux 内核为每个分配阶维护一个空闲页面列表，并且 `get_free_pages` 只会增加簇中第一个页面的引用计数，`free_pages` 则会减少该计数。如果分配阶大于 0，`scullp` 设备的 `mmap` 方法将被禁用，因为 `nopage` 处理的是单个页面，而不是页面簇。`scullp` 根本不知道如何正确管理高阶分配中页面的引用计数。（如果需要复习 `scullp` 和内存分配阶的值，请返回 8.3.1 节。）

零阶限制主要是为了简化代码。通过处理页面的使用计数，可以正确实现多页面分配的 `mmap`，但这只会增加示例的复杂性，而不会引入任何有价值的信息。

根据上述规则映射 RAM 的代码需要实现 `open`、`close` 和 `nopage` VMA 方法；还需要访问内存映射来调整页面使用计数。

`scullp_mmap` 的实现非常简短，因为它依赖 `nopage` 函数来完成所有重要的工作：
```c
int scullp_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct inode *inode = filp->f_dentry->d_inode;

    /* 如果阶数不为 0，则拒绝映射 */
    if (scullp_devices[iminor(inode)].order)
        return -ENODEV;

    /* 这里不做任何事情："nopage" 会填补空白 */
    vma->vm_ops = &scullp_vm_ops;
    vma->vm_flags |= VM_RESERVED;
    vma->vm_private_data = filp->private_data;
    scullp_vma_open(vma);
    return 0;
}
```
`if` 语句的目的是避免映射分配阶数不为 0 的设备。`scullp` 的操作存储在 `vm_ops` 字段中，设备结构的指针存储在 `vm_private_data` 字段中。最后，调用 `vm_ops->open` 来更新设备的活动映射计数。

`open` 和 `close` 方法只是跟踪映射计数，定义如下：
```c
void scullp_vma_open(struct vm_area_struct *vma)
{
    struct scullp_dev *dev = vma->vm_private_data;

    dev->vmas++;
}

void scullp_vma_close(struct vm_area_struct *vma)
{
    struct scullp_dev *dev = vma->vm_private_data;

    dev->vmas--;
}
```
大部分工作由 `nopage` 方法完成。在 `scullp` 的实现中，`nopage` 的 `address` 参数用于计算设备的偏移量；然后使用该偏移量在 `scullp` 内存树中查找正确的页面：
```c
struct page *scullp_vma_nopage(struct vm_area_struct *vma,
                                unsigned long address, int *type)
{
    unsigned long offset;
    struct scullp_dev *ptr, *dev = vma->vm_private_data;
    struct page *page = NOPAGE_SIGBUS;
    void *pageptr = NULL; /* 默认 "缺失" */

    down(&dev->sem);
    offset = (address - vma->vm_start) + (vma->vm_pgoff << PAGE_SHIFT);
    if (offset >= dev->size) goto out; /* 超出范围 */

    /*
     * 现在从列表中检索 scullp 设备，然后是页面。
     * 如果设备有空洞，进程访问空洞时会收到 SIGBUS 信号。
     */
    offset >>= PAGE_SHIFT; /* 偏移量是页面数 */
    for (ptr = dev; ptr && offset >= dev->qset;) {
        ptr = ptr->next;
        offset -= dev->qset;
    }
    if (ptr && ptr->data) pageptr = ptr->data[offset];
    if (!pageptr) goto out; /* 空洞或文件末尾 */
    page = virt_to_page(pageptr);
    
    /* 找到页面，现在增加计数 */
    get_page(page);
    if (type)
        *type = VM_FAULT_MINOR;
out:
    up(&dev->sem);
    return page;
}
```

`scullp` 使用通过 `get_free_pages` 函数获取的内存。这些内存通过逻辑地址来访问，所以 `scullp_nopage` 函数若要获取一个 `struct page` 指针，只需调用 `virt_to_page` 函数即可。

现在，`scullp` 设备能够如预期般工作，从 `mapper` 工具的示例输出中便可看出。下面，我们将 `/dev` 目录的列表信息（内容较多）发送到 `scullp` 设备，然后使用 `mapper` 工具通过 `mmap` 来查看该列表的部分内容：
```plaintext
morgana% ls -l /dev > /dev/scullp
morgana% ./mapper /dev/scullp 0 140
已将 "/dev/scullp" 从 0 (0x00000000) 映射到 140 (0x0000008c)
总计 232
crw-------    1 root     root      10,  10 9月 15 07:40 adbmouse
crw-r--r--    1 root     root      10, 175 9月 15 07:40 agpgart
morgana% ./mapper /dev/scullp 8192 200
已将 "/dev/scullp" 从 8192 (0x00002000) 映射到 8392 (0x000020c8)
d0h1494
brw-rw----    1 root     floppy     2,  92 9月 15 07:40 fd0h1660
brw-rw----    1 root     floppy     2,  20 9月 15 07:40 fd0h360
brw-rw----    1 root     floppy     2,  12 9月 15 07:40 fd0H360
```

## 15.2.7 重映射内核虚拟地址
尽管这种需求很少见，但了解驱动程序如何使用 `mmap` 将内核虚拟地址映射到用户空间还是很有趣的。请记住，真正的内核虚拟地址是像 `vmalloc` 这样的函数返回的地址，也就是在内核页表中映射的虚拟地址。本节的代码取自 `scullv` 模块，该模块的功能与 `scullp` 类似，但它通过 `vmalloc` 来分配存储空间。

`scullv` 的大部分实现与我们刚刚看到的 `scullp` 实现类似，不同之处在于无需检查控制内存分配的 `order` 参数。这是因为 `vmalloc` 每次只分配一个页面，因为单页分配比多页分配成功的可能性要大得多。因此，分配阶数的问题在 `vmalloc` 分配的空间中并不存在。

除此之外，`scullp` 和 `scullv` 所使用的 `nopage` 实现之间只有一处不同。回顾一下，`scullp` 在找到所需页面后，会使用 `virt_to_page` 函数来获取对应的 `struct page` 指针。然而，这个函数对内核虚拟地址并不适用。相反，你必须使用 `vmalloc_to_page` 函数。因此，`scullv` 版本的 `nopage` 函数的最后部分代码如下所示：
```c
  /*
   * 在 scullv 查找之后，"page" 现在是当前进程所需页面的地址。
   * 由于这是一个 vmalloc 地址，需要将其转换为 struct page。
   */
  page = vmalloc_to_page(pageptr);
    
  /* 找到了，现在增加引用计数 */
  get_page(page);
  if (type)
      *type = VM_FAULT_MINOR;
out:
  up(&dev->sem);
  return page;
```
基于以上讨论，你可能还想将 `ioremap` 返回的地址映射到用户空间。然而，这是一个错误的做法；`ioremap` 返回的地址很特殊，不能像处理普通内核虚拟地址那样处理它们。相反，你应该使用 `remap_pfn_range` 函数将 I/O 内存区域重映射到用户空间。 