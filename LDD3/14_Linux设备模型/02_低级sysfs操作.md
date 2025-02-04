# 14.2 底层 Sysfs 操作
`kobject` 是 `sysfs` 虚拟文件系统背后的机制。对于 `sysfs` 中的每个目录，在内核的某个地方都有一个 `kobject` 与之对应。每个受关注的 `kobject` 还会导出一个或多个属性，这些属性在该 `kobject` 的 `sysfs` 目录中以包含内核生成信息的文件形式出现。本节将深入探讨 `kobject` 和 `sysfs` 在底层是如何交互的。

处理 `sysfs` 的代码应该包含 `<linux/sysfs.h>` 头文件。

要让一个 `kobject` 在 `sysfs` 中显示，只需调用 `kobject_add` 函数即可。我们之前已经了解到，这个函数可用于将 `kobject` 添加到 `kset` 中，而在 `sysfs` 中创建条目也是它的工作之一。关于 `sysfs` 条目是如何创建的，有几点值得了解：
- **总是创建目录**：`kobject` 在 `sysfs` 中的条目总是以目录形式存在，因此调用 `kobject_add` 会在 `sysfs` 中创建一个目录。通常，这个目录会包含一个或多个属性，我们稍后会介绍如何指定这些属性。
- **名称一致性**：使用 `kobject_set_name` 为 `kobject` 分配的名称，就是 `sysfs` 目录所使用的名称。因此，在 `sysfs` 层次结构的同一部分中出现的 `kobject` 必须具有唯一的名称。分配给 `kobject` 的名称也应该是合理的文件名：不能包含斜杠字符，并且强烈不建议使用空格。
- **父目录定位**：`sysfs` 条目位于与 `kobject` 的 `parent` 指针相对应的目录中。如果在调用 `kobject_add` 时 `parent` 为 `NULL`，它会被设置为新 `kobject` 所在 `kset` 中嵌入的 `kobject`；因此，`sysfs` 层次结构通常与使用 `kset` 创建的内部层次结构相匹配。如果 `parent` 和 `kset` 都为 `NULL`，则会在顶级创建 `sysfs` 目录，这几乎肯定不是你想要的结果。

使用我们目前所描述的机制，我们可以使用 `kobject` 在 `sysfs` 中创建一个空目录。通常，你可能希望做一些更有意义的事情，所以现在是时候看看属性的实现了。

## 14.2.1 默认属性
每个 `kobject` 在创建时都会被赋予一组默认属性。这些属性是通过 `kobj_type` 结构体来指定的。请记住，该结构体的定义如下：
```c
struct kobj_type {
    void (*release)(struct kobject *);
    struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
};
```
`default_attrs` 字段列出了要为该类型的每个 `kobject` 创建的属性，而 `sysfs_ops` 则提供了实现这些属性的方法。我们先从 `default_attrs` 开始，它指向一个指向属性结构体的指针数组：
```c
struct attribute {
    char *name;
    struct module *owner;
    mode_t mode;
};
```
在这个结构体中，`name` 是属性的名称（在 `kobject` 的 `sysfs` 目录中显示的名称），`owner` 是指向负责实现该属性的模块的指针（如果有的话），`mode` 是要应用于该属性的保护位。对于只读属性，`mode` 通常设置为 `S_IRUGO`；如果该属性是可写的，可以加上 `S_IWUSR` 以仅给予 root 用户写访问权限（模式的宏定义在 `<linux/stat.h>` 中）。`default_attrs` 列表的最后一个条目必须全部置为 0。

`default_attrs` 数组说明了属性是什么，但并没有告诉 `sysfs` 如何实际实现这些属性。这项任务由 `kobj_type->sysfs_ops` 字段完成，它指向一个定义如下的结构体：
```c
struct sysfs_ops {
    ssize_t (*show)(struct kobject *kobj, struct attribute *attr, 
                    char *buffer);
    ssize_t (*store)(struct kobject *kobj, struct attribute *attr, 
                     const char *buffer, size_t size);
};
```
每当从用户空间读取一个属性时，`show` 方法会被调用，同时传入指向 `kobject` 的指针和相应的属性结构体。该方法应将给定属性的值编码到 `buffer` 中，要确保不超出其大小（为 `PAGE_SIZE` 字节），并返回实际返回数据的长度。`sysfs` 的约定规定，每个属性应包含一个单一的、人类可读的值；如果要返回大量信息，你可能需要考虑将其拆分为多个属性。

同一个 `show` 方法会用于处理与给定 `kobject` 关联的所有属性。传入该函数的 `attr` 指针可用于确定请求的是哪个属性。一些 `show` 方法会对属性名进行一系列测试。其他实现方式则将属性结构体嵌入到另一个包含返回属性值所需信息的结构体中；在这种情况下，可在 `show` 方法中使用 `container_of` 来获取指向嵌入结构体的指针。

`store` 方法与之类似；它应解码存储在 `buffer` 中的数据（`size` 包含该数据的长度，不超过 `PAGE_SIZE`），以合理的方式存储并响应新值，并返回实际解码的字节数。只有当属性的权限允许写入时，才会调用 `store` 方法。编写 `store` 方法时，永远不要忘记你接收到的是来自用户空间的任意信息；在采取任何响应行动之前，你应该非常仔细地验证它。如果传入的数据不符合预期，应返回一个负的错误值，而不是可能执行一些不想要且不可恢复的操作。如果你的设备导出了一个 `self_destruct` 属性，你应该要求写入一个特定的字符串才能触发该功能；意外的随机写入应仅返回一个错误。

## 14.2.2 非默认属性
在很多情况下，`kobject` 类型的 `default_attrs` 字段描述了该 `kobject` 所有的属性。但这并非设计上的限制；可以随意为 `kobject` 添加和移除属性。如果你想为 `kobject` 的 `sysfs` 目录添加一个新属性，只需填充一个属性结构体并将其传递给以下函数：
```c
int sysfs_create_file(struct kobject *kobj, struct attribute *attr);
```
如果一切顺利，将以属性结构体中给定的名称创建文件，返回值为 0；否则，将返回通常的负错误码。

请注意，实现新属性的操作时会调用相同的 `show()` 和 `store()` 函数。在为 `kobject` 添加新的非默认属性之前，你应该采取必要的步骤，确保这些函数知道如何实现该属性。

要移除一个属性，调用：
```c
int sysfs_remove_file(struct kobject *kobj, struct attribute *attr);
```
调用之后，该属性将不再出现在 `kobject` 的 `sysfs` 条目中。不过要注意，用户空间进程可能持有该属性的打开文件描述符，并且在属性被移除后，仍然可以调用 `show` 和 `store` 方法。

## 14.2.3 二进制属性
`sysfs` 约定要求所有属性都以人类可读的文本格式包含单个值。不过，偶尔会有创建能够处理较大二进制数据块的属性的需求。这种需求通常出现在数据必须在用户空间和设备之间原封不动地传递时。例如，向设备上传固件就需要这个功能。当系统中遇到这样的设备时，可以启动一个用户空间程序（通过热插拔机制）；该程序随后通过一个二进制 `sysfs` 属性将固件代码传递给内核，如 14.8.1 节所示。

二进制属性由 `bin_attribute` 结构体描述：
```c
struct bin_attribute {
    struct attribute attr;
    size_t size;
    ssize_t (*read)(struct kobject *kobj, char *buffer, 
                    loff_t pos, size_t size);
    ssize_t (*write)(struct kobject *kobj, char *buffer, 
                    loff_t pos, size_t size);
};
```
这里，`attr` 是一个属性结构体，用于指定二进制属性的名称、所属模块以及权限；`size` 表示该二进制属性的最大大小（若没有最大限制，则设为 0）。`read` 和 `write` 方法的工作方式与普通字符设备驱动里对应的方法类似；对于一次数据加载操作，它们可能会被多次调用，每次调用最多能处理一页大小的数据。由于 `sysfs` 没有机制来标识一组写操作中的最后一次，所以实现二进制属性的代码必须通过其他方式来判断数据传输是否结束。

二进制属性必须显式创建，不能将其设置为默认属性。若要创建一个二进制属性，可调用如下函数：
```c
int sysfs_create_bin_file(struct kobject *kobj, 
                          struct bin_attribute *attr);
```

若要移除二进制属性，可使用以下函数：
```c
int sysfs_remove_bin_file(struct kobject *kobj, 
                          struct bin_attribute *attr);
```

## 14.2.4 符号链接
`sysfs` 文件系统采用常见的树形结构，反映了它所表示的 `kobject` 的层次化组织方式。然而，内核中对象之间的关系往往更为复杂。例如，`sysfs` 中的一个子树（`/sys/devices`）代表系统中已知的所有设备，而其他子树（位于 `/sys/bus` 下）则代表设备驱动。但这些树状结构并未体现出驱动与其管理的设备之间的关系。若要展示这些额外的关系，就需要额外的指针，在 `sysfs` 中，这是通过符号链接来实现的。

在 `sysfs` 中创建符号链接很简单：
```c
int sysfs_create_link(struct kobject *kobj, struct kobject *target,
                      char *name);
```
此函数会创建一个名为 `name` 的链接，它指向 `target` 的 `sysfs` 条目，并将其作为 `kobj` 的一个属性。这是一个相对链接，因此无论 `sysfs` 在特定系统上挂载在何处，它都能正常工作。

即使 `target` 从系统中移除，该链接仍然会保留。如果你要创建指向其他 `kobject` 的符号链接，或许应该有办法了解这些 `kobject` 的变化情况，或者能确保目标 `kobject` 不会消失。虽然出现死符号链接（`sysfs` 中的无效符号链接）的后果并不严重，但这并非最佳编程实践，还可能会在用户空间引发混淆。

若要移除符号链接，可使用以下函数：
```c
void sysfs_remove_link(struct kobject *kobj, char *name);
``` 