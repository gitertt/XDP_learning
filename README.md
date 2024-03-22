# XDP_learning
XDP学习记录
### XDP作用位置

**网络数据包处理流程**：
当一个网络数据包到达设备时，它会经过一系列复杂的处理步骤，这些步骤涉及硬件（如网卡）和软件（如操作系统内核中的网络堆栈）。以下是一个数据包从到达设备到被处理的典型过程：

1. 物理接收

**网卡收到数据包**：数据包通过物理介质（如以太网电缆、光纤或无线信号）到达网卡。网卡的硬件负责检测和接收这些电子或光学信号，并将其转换为可以被网卡处理器理解的数字信息。

2. 硬件处理

**DMA传输**：网卡使用直接内存访问（DMA）将数据包从网卡缓冲区传输到主机内存中的预定义缓冲区。这个过程不需要CPU的干预，可以提高数据传输的效率。

——**<u>XDP作用位置</u>**——

3. 中断

**触发中断**：一旦数据包被复制到主机内存，网卡会向CPU发送一个中断信号，告知CPU有新的数据包到达。

4. 内核处理

**中断处理**：CPU接收到中断后，会中断当前的任务，转而执行网络中断处理程序。这个程序是操作系统内核的一部分，负责处理网络事件。

**数据包接收处理**：中断处理程序会进行初步的数据包处理，这可能包括错误检查、数据包过滤（如通过防火墙规则）等。

**软中断/软件队列**：为了提高效率，一些进一步的数据包处理可能会被推迟到软中断或网络软件队列中处理。这允许CPU快速返回到其他任务，而不是花费太多时间在单个数据包上。

5. 协议栈处理

**协议解析**：数据包会经过操作系统内核的网络协议栈，栈中的每一层（如链路层、网络层、传输层）会解析相应层的头部信息，并根据需要进行处理。例如，IP层会检查数据包的目的IP地址，TCP层会检查端口号。

**路由判断**：如果设备配置为路由器或具有路由功能，内核还会根据路由表判断数据包的下一跳地址。

6. 交付给应用

**套接字缓冲区**：最终，数据包到达传输层（如TCP或UDP）后，会根据端口号被交付给对应的套接字缓冲区。应用程序通过读取这些套接字来获取数据包内容。

7. 应用程序处理

**应用逻辑**：应用程序从套接字中读取数据，然后根据应用逻辑进行处理。这可能涉及数据库操作、消息响应、或者进一步的数据转发等。

这个过程涉及到了从物理层到应用层的多个层次，每一层都有自己的职责和处理逻辑。高效的网络数据包处理是现代计算设备网络性能的关键组成部分。



### 1.加载XDP程序

(1)首先将XDP的受限C程序编译为BPF字节流，并存储在ELF目标文件中：

**clang -O2 <u>-target bpf</u> -c xdp.c -o xdp.o**

(2)您可以使用不同的工具（例如 readelf 或 **llvm-objdump**）检查 xdp.o 文件的内容

命令：**llvm-objdump -S xdp.o**

(3)通过**iproute2 ip**加载XDP程序

命令：**sudo ip link set dev lo xdpgeneric obj xdp_pass_kern.o sec xdp**

其中，**lo**为网络设备端口名称；（一般不采用lo）

**xdpgeneric**为指定要使用 XDP 的通用模式；

从端口处移除XDP程序命令：**sudo ip link set dev lo xdpgeneric off**

**注意！！！**`ip` 命令不支持 XDP 多分发协议，即只能在端口处添加一个XDP程序，若添加多个，则只有最新添加的XDP程序有效。

以下为ip常用操作：

`iproute2` 是 Linux 下一套强大的网络工具集，用于网络设备和链接管理、路由表管理等。它是传统网络工具（如 `ifconfig`、`route`、`netstat` 等）的现代替代品，提供了更丰富的功能和更细致的控制能力。以下是一些使用 `iproute2` 的常用操作示例：

a.**显示网络接口信息**

要查看所有网络接口的信息，可以使用：

```bash
ip link show
```

或简写为：

```bash
ip l
```

b.**启用和禁用网络接口**

启用名为 `eth0` 的网络接口：

```bash
ip link set eth0 up
```

禁用名为 `eth0` 的网络接口：

```bash
ip link set eth0 down
```

**c.配置和显示IP地址**

为名为 `eth0` 的网络接口分配 IP 地址：

```bash
ip addr add 192.168.1.100/24 dev eth0
```

查看所有接口的 IP 地址：

```bash
ip addr show
```

或简写为：

```bash
ip a
```

**d.查看和修改路由表**

查看路由表：

```bash
ip route show
```

或简写为：

```bash
ip r
```

添加一条到达 `192.168.2.0/24` 网络的路由，通过网关 `192.168.1.1`：

```bash
ip route add 192.168.2.0/24 via 192.168.1.1
```

删除上述路由：

```bash
ip route del 192.168.2.0/24
```

(4)使用xdp-loader来加载XDP程序

命令：**sudo xdp-loader load -m skb lo xdp_pass_kern.o**

其中，

- `xdp-loader`：是用于加载和管理 XDP 程序的工具。它是 libbpf 工具集的一部分，提供了一个高级接口来操作 eBPF/XDP 程序。
- `load`：是 `xdp-loader` 工具的一个子命令，用于加载 XDP 程序到指定的网络接口。
- `-m skb`：指定加载模式为 SKB（Socket Buffer）模式，也被称为通用 XDP（Generic XDP）。这种模式与驱动程序无关，兼容性更好，但性能不如原生模式（native mode）。在 SKB 模式下，XDP 程序作用于软件网络堆栈层面，而不是在网络驱动层面。

### 2.加载BPF ELF文件中的特定XDP程序

(1)设置测试环境

利用已有脚本生成。

创建一个测试环境：./testenv.sh setup --name=test（设置了别名后：testenv  setup --name=test)

查看测试环境状态：testenv status

(2)通过函数名加载特定XDP程序

使用xdp-loader工具可以在同一个网络接口加载多个XDP程序。当网络数据到达时，会按优先级依次执行XDP程序。

### 3.使用BFP maps计数

(1)定义一个map

map的四要素：map类型、键、值、允许存储的最大键值对数量。

例如：

```
struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__type(key, __u32);
	__type(value, struct datarec);
	__uint(max_entries, XDP_ACTION_MAX);
} xdp_stats_map SEC(".maps");
```

(2)上下文变量**xdp_md**的解析

```
struct xdp_md {
	// (Note: type __u32 is NOT the real-type)
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	/* Below access go through struct xdp_rxq_info */
	__u32 ingress_ifindex; /* rxq->dev->ifindex */
	__u32 rx_queue_index;  /* rxq->queue_index */
};
```

在 XDP (eXpress Data Path) 程序中，当内核调用 eBPF 程序时，它会传递一个指向 `struct xdp_md` 类型的上下文变量 (`ctx`)。这个结构体提供了关于正在处理的数据包的信息。然而，`struct xdp_md` 的定义可能会让人困惑，因为它的所有成员都是 `__u32` 类型，这并不代表它们的实际数据类型。当 eBPF 程序加载到内核时，对这个数据结构的访问会被内核映射（remapped）到实际的数据结构上，如 `struct xdp_buff` 和 `struct xdp_rxq_info`。

**为什么要设置数据结构的映射？**

a.抽象与简化的作用。通过提供一个标准化和简化的接口（如 `xdp_md` 结构），eBPF 程序能够以一种高度抽象的方式编写，而无需关心底层数据结构的复杂性和变化。这样做简化了 eBPF 程序的开发，使得程序更容易理解和维护。

b.兼容性和灵活性。这样设置的话，当内核中数据结构发生变化时，只要保持该映射，就不用改变现有eBPF程序。提供了一种向后兼容的方式，使得 eBPF 程序即使在内核更新后仍能正常运行。



xdp_md中各变量解释：

- `data` 和 `data_end` 是数据包内容在内存中的起始和结束位置的指针（以 __u32 类型表示，但实际上是内存地址）。
- `data_meta` 提供了与数据包元数据相关的信息（如果有的话）。

`data_meta` 字段是 `xdp_md` 结构的一部分，用于指向数据包的元数据的开始位置。元数据是与数据包相关联的，但不是数据包内容的一部分的信息。在 XDP 程序中，元数据可以用于传递关于数据包的额外信息，比如自定义标签或者标识符。

`data_meta` 可以用于多种场景，其中一些示例包括：

**自定义数据包标记**：可以使用元数据字段来标记数据包，例如，为了指示数据包应该由特定的 eBPF 程序处理，或者标记数据包的优先级。

**传递信息**：在复杂的 eBPF 应用中，不同的 eBPF 程序可能需要协同工作。`data_meta` 可以用来在这些程序之间传递信息。

**性能优化**：通过在元数据中存储状态信息或其他指标，可以减少对 eBPF maps 的访问，从而优化性能。

- `ingress_ifindex` 和 `rx_queue_index` 分别表示接收数据包的网络接口索引和接收队列索引，这些访问是通过 `struct xdp_rxq_info` 进行的。

在 XDP 程序中，`ingress_ifindex` 可以**用来区分数据包是从哪个网络接口进入的**。这对于多网卡环境中的流量过滤和统计尤其有用。例如，你可能想要针对从特定网络接口接收到的数据包应用不同的策略或进行特定的统计。

代码示例研究：

```
/* SPDX-License-Identifier: GPL-2.0 */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

#include "common_kern_user.h" /* defines: struct datarec; */

/* Lesson: See how a map is defined.
 * - Here an array with XDP_ACTION_MAX (max_)entries are created.
 * - The idea is to keep stats per (enum) xdp_action
 */
struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__type(key, __u32);
	__type(value, struct datarec);
	__uint(max_entries, XDP_ACTION_MAX);
} xdp_stats_map SEC(".maps");

/* LLVM maps __sync_fetch_and_add() as a built-in function to the BPF atomic add
 * instruction (that is BPF_STX | BPF_XADD | BPF_W for word sizes)
 */
#ifndef lock_xadd
#define lock_xadd(ptr, val)	((void) __sync_fetch_and_add(ptr, val))
#endif

static __always_inline
__u32 xdp_stats_record_action(struct xdp_md *ctx, __u32 action)
{
	void *data_end = (void *)(long)ctx->data_end;
	void *data     = (void *)(long)ctx->data;

	if (action >= XDP_ACTION_MAX)
		return XDP_ABORTED;

	/* Lookup in kernel BPF-side return pointer to actual data record */
	struct datarec *rec = bpf_map_lookup_elem(&xdp_stats_map, &action);
	if (!rec)
		return XDP_ABORTED;

	/* Calculate packet length */
	__u64 bytes = data_end - data;

	/* BPF_MAP_TYPE_PERCPU_ARRAY returns a data record specific to current
	 * CPU and XDP hooks runs under Softirq, which makes it safe to update
	 * without atomic operations.
	 */
	rec->rx_packets++;
	rec->rx_bytes += bytes;

	return action;
}

SEC("xdp")
int  xdp_pass_func(struct xdp_md *ctx)
{
	__u32 action = XDP_PASS; /* XDP_PASS = 2 */

	return xdp_stats_record_action(ctx, action);
}

SEC("xdp")
int  xdp_drop_func(struct xdp_md *ctx)
{
	__u32 action = XDP_DROP;

	return xdp_stats_record_action(ctx, action);
}

SEC("xdp")
int  xdp_abort_func(struct xdp_md *ctx)
{
	__u32 action = XDP_ABORTED;

	return xdp_stats_record_action(ctx, action);
}

char _license[] SEC("license") = "GPL";

/* Copied from: $KERNEL/include/uapi/linux/bpf.h
 *
 * User return codes for XDP prog type.
 * A valid XDP program must return one of these defined values. All other
 * return codes are reserved for future use. Unknown return codes will
 * result in packet drops and a warning via bpf_warn_invalid_xdp_action().
 *
enum xdp_action {
	XDP_ABORTED = 0,
	XDP_DROP,
	XDP_PASS,
	XDP_TX,
	XDP_REDIRECT,
};

 * user accessible metadata for XDP packet hook
 * new fields must be added to the end of this structure
 *
struct xdp_md {
	// (Note: type __u32 is NOT the real-type)
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	// Below access go through struct xdp_rxq_info
	__u32 ingress_ifindex; // rxq->dev->ifindex
	__u32 rx_queue_index;  // rxq->queue_index
};
*/
```
