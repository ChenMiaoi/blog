# RXE Detail

---  

在RDMA的发展中，我们不可避免地会遇到这样的一个问题：*我没有硬件，但我还是想要使用RDMA功能进行测试或者学习*。对于这个需求，RDMA社区实现了一套由软件模拟硬件的方案：RXE(Soft-RoCE)。

[RXE(RDMA over Converged Ethernet)](https://github.com/SoftRoCE)**通过标准的以太网设施提供RDMA功能，不需要专用的RDMA硬件**。RXE是Linux内核中的一个驱动程序，允许常规以太网网卡模拟RDMA功能，从而支持RDMA应用程序在廉价的硬件上运行。

因此，本章节的主要目标便是对RXE如何模拟出一个真实的RDMA硬件做出分析。

## How To Monitor a RDMA-like Device

在探讨RXE如何模拟RDMA协议和硬件之前，我们需要知道：RDMA是需要有专门的网卡设施的。因此，本节的主要目标是：**分析RXE如何模拟出一个RDMA网卡设备的**。

开始之前，我们需要启用Linux内核中的RXE驱动模块：

<code-block lang="bash">
cat /boot/config-* | grep RXE
CONFIG_RDMA_RXE=m
</code-block>

如上面的代码所示，如果查看内核的配置文件，我们就能够发现CONFIG_RDMA_RXE的存在。我们需要启用该驱动程序：

<code-block lang="Bash">
sudo modprobe rdma_rxe
</code-block>

> How to Enable a Network card be a RDMA device  
> 在2020年以前，我们可以通过rxe_cfg进行配置。2020年以后，rxe_cfg实际被[rdma](https://github.com/iproute2/iproute2/tree/main/rdma)命令所替代。
> <code-block lang="bash">rdma link add [device_name] type rxe netdev [network_device]</code-block>

### Basic Structure

通过rdma命令创建RDMA网络设备，实际上是通过Netlink内核模块与RDMA子系统进行交互，完成设备配置、状态查询和调试任务。

因此，在[rdma.h](https://github.com/iproute2/iproute2/blob/main/rdma/rdma.h)中，使用了一些结构体用于管理RDMA设备信息。

#### struct rd

[struct rd](https://github.com/iproute2/iproute2/blob/0325d98f98baebd27cf7bd7f68b1e9eba4dc7d5b/rdma/rdma.h#L57)是RDMA配置的核心数据结构，整合了设备列表、过滤器以及Netlink通信相关信息：  

<code-block lang="c++">
struct rd {
	int                argc;            // 由命令行传入的参数数量
	char**             argv;            // 由命令行传入的参数列表
	char*              filename;
	struct list_head   dev_map_list;    // 使用双向链表管理多个设备信息
	uint32_t           dev_idx;         // 当前操作的设备索引
	uint32_t           port_idx;        // 当前操作的RDMA端口索引
	struct mnl_socket* nl;              // Netlink套接字，用于与内核通信，建立连接等操作
	struct nlmsghdr*   nlh;             // Netlink的消息头，用于通信时构造和解析Netlink数据包
	char*              buff;            // 存储Netlink的数据
	struct list_head   filter_list;     // 
	char*              link_name;       // 
	char*              link_type;       // 描述RDMA设备的通信方式(RoCE or Infiniband)
	char*              dev_name;        // 当前操作的设备名称
	char*              dev_type;        // 当前操作的设备类型
} rd;
</code-block>

#### struct filter_entry

#### struct dev_map

### RDMA init

首先，我们先从宏观上来观察rd_init的作用。从下面的甘特图可以得知，rd_init会进行一系列初始化后，构建出一个Netlink需要的数据报头，发送请求给内核，然后再接受从内核返回的数据信息。

``` mermaid
gantt
    dateFormat s
    axisFormat %S
    section rd_init
        init dev_map_list       :a1, 0, 10s
    section init_detail
        rd_init_start           :milestone, m1, 0, 0
        rd_prepare_msg          :a1, after m1, 2
        rd_send_msg             :a2, after a1, 6
        rd_recv_msg             :a3, after a2, 10
        rd_init_end             :milestone, m2, 10, 10
```

我们先从`rd_send_msg`之前的操作进行分析。

#### Before rd_send_msg

在发送之前自然就是初始化操作，以及准备send所需要的数据空间，以便构建数据包发送给Netlink。

<code-block lang="C++">
INIT_LIST_HEAD( &rd->dev_map_list );
INIT_LIST_HEAD( &rd->filter_list );

rd->buff = (char*) malloc( MNL_SOCKET_BUFFER_SIZE );
</code-block>

初始化完Netlink报文存放的缓冲区后，我们就可以开始准备报文信息了：

<code-block lang="C++">
void rd_prepare_msg( struct rd* rd,     //  
                     uint32_t cmd,      //
                     uint32_t* seq,     //
                     uint16_t flags     // 
) {
    *seq = time( NULL );

	rd->nlh              = mnl_nlmsg_put_header( rd->buff );
	rd->nlh->nlmsg_type  = RDMA_NL_GET_TYPE( RDMA_NL_NLDEV, cmd );
	rd->nlh->nlmsg_seq   = *seq;
	rd->nlh->nlmsg_flags = flags;
}

rd_prepare_msg( rd, RDMA_NLDEV_CMD_GET, &seq,
                ( NLM_F_REQUEST | NLM_F_ACK | NLM_F_DUMP ) );
</code-block>

在内核中，RDMA子系统会为Netlink注册一个客户端设备以响应各种请求([nldev.c](https://github.com/torvalds/linux/blob/98f7e32f20d28ec452afb208f9cffc08448a2652/drivers/infiniband/core/nldev.c#L2727))。在内核代码中，我们可以清晰的看见，`RDMA_NL_NLDEV`就是该客户端的注册索引，其对应了一系列的回调函数：

<code-block lang="C++">
rdma_nl_register(RDMA_NL_NLDEV, nldev_cb_table);

static const struct rdma_nl_cbs nldev_cb_table[RDMA_NLDEV_NUM_OPS] = {
    [RDMA_NLDEV_CMD_GET] = {
        .doit = nldev_get_doit,
        .dump = nldev_get_dumpit,
    },
}
</code-block>

而`RDMA_NLDEV_CMD_GET`就是其中的一个回调操作，执行两个函数：

- `nldev_get_doit`通常用于查询单个RDMA设备信息
  - 解析用户态发送的Netlink报文信息
  - 根据索引查找对应的RDMA设备
  - 构建新的Netlink报文信息，并填充找到的RDMA设备信息
  - 响应用户态信息
- `nldev_get_dumpit`通常用于转存所有RDMA设备信息

现在回过头来分析`flag`字段：

- NLM_F_REQUEST
  - 表示这是由用户态发起的请求
- NML_F_ACK
  - 表示对端收到请求后，需要响应一个ACK消息
  - 这也就是后续`rd_send_msg`后使用`rd_recv_msg`的原因
- NML_F_DUMP
  - 表示请求列举或获取一个资源的完整列表，而不是查询单一的资源。
  - 结合上面的`nldev_get_dumpit`，这里会使得消息发送到RDMA子系统时执行`nldev_get_dumpit`操作而非`nldev_get_doit`。

至此，`rd_prepare_msg`就分析完毕，现在我们来详细分析`rd_send_msg`。

#### rd_send_msg

``` mermaid
gantt
    dateFormat s
    axisFormat %S
    section rd_send
        rd_send_msg     :a1, 0, 15s
    section detail
        start send      :milestone m1, 0, 0
        end send        :milestone m2, 15, 15
        mnlu_socket_open        :a4, 0, 9
        mnl_socket_sendto       :a5, 9, 15
        mnl_socket_open         :a1, after m1, 3
        mnl_socket_setsockopt   :a2, after a1, 6
        mnl_socket_bind         :a3, after a2, 9
```

上图是关于`rd_send_msg`的全部流程。首先我们先从内核中进行分析，在RDMA子系统的[netlink.c](https://github.com/torvalds/linux/blob/98f7e32f20d28ec452afb208f9cffc08448a2652/drivers/infiniband/core/netlink.c#L317)中会对创建的`netlink`设备进行设置：

<code-block lang="C++">
struct sock *nls;
nls = netlink_kernel_create(net, NETLINK_RDMA, &cfg);
</code-block>

因此，我们的客户端想要对RDMA子系统发送请求，就需要率先与RDMA子系统建立连接。

<code-block lang="C++">
rd->nl = mnlu_socket_open( NETLINK_RDMA );
{
    nl = mnl_socket_open( NETLINK_RDMA );
    if ( nl == NULL ) return NULL;

    mnl_socket_setsockopt( nl, NETLINK_CAP_ACK, &one, sizeof( one ) );
    mnl_socket_setsockopt( nl, NETLINK_EXT_ACK, &one, sizeof( one ) );
    
    if ( mnl_socket_bind( nl, 0, MNL_SOCKET_AUTOPID ), 0 ) goto err_bind;
}
</code-block>

在之前我们准备`Netlink`报文的时候了解到，我们需要内核返回一个ACK响应信息。因此，我们需要对`mnl`进行配置：

- NETLINK_CAP_ACK(Capability)
  - 该标志要求内核在接收到请求时返回确认消息
- NETLINK_EXT_ACK(Extended)
  - 启用扩展确认机制，扩展确认不仅仅包含请求是否成功的信息，还可能包含更多的错误信息和调试信息

设置完`Netlink`套接字的配置后，我们就能够进行绑定了，此处我们设置了自动绑定进程ID以及自动由绑定的PID进行分组。

建立连接后，我们就可以对打开的`Netlink`套接字进行传输消息：

<code-block lang="C++">
ret = mnl_socket_sendto( rd->nl, rd->nlh, rd->nlh->nlmsg_len );
</code-block>

发送完消息后，内核中的`nldev`就会根据解析出的`Netlink`报文信息执行对应的操作。至此，`rd_send_msg`分析完毕。

#### rd_recv_msg

``` mermaid
gantt
    dateFormat s
    axisFormat %S
    section rd_recv
        rd_recv_msg     :a1, 0, 15s
    section detail
        start recv      :milestone m1, 0, 0
        end recv        :milestone m2, 15, 15
        mnl_socket_recvfrom     :a2, after m1, 3
        mnl_cb_run2             :a3, after a2, 15
        
        start rd_dev_init_cb    :milestome m3, 3, 3
        end rd_dev_init_cb      :milestome m4, 15, 15
        mnl_attr_parse          :a4, after a2, 6
        list_add_tail           :a5, after a4, 15
```

从上面的甘特图可以看见，实际上`rd_recv_msg`对于接受返回的报文信息十分简单，只需要调用`mnl_socket_recvfrom`即可。之后的所有操作都是解析返回的报文，并将含有的RDMA设备信息添加到设备列表中。

对于处理`mnl_socket_recvfrom`收到的报文，我们可以通过使用`mnl_cb_run2`进行分析，`mnl_cb_run2`会验证响应报文的正确性，并合理调用合适的回调函数：

``` c++
static inline int __mnl_cb_run( const void* buf, size_t numbytes,
                                unsigned int seq, unsigned int portid,
                                mnl_cb_t cb_data, void* data,
                                const mnl_cb_t* cb_ctl_array,
                                unsigned int    cb_ctl_array_len );

if ( nlh->nlmsg_type >= NLMSG_MIN_TYPE ) {
    if ( cb_data ) {
        ret = cb_data( nlh, data );
        if ( ret <= MNL_CB_STOP )
            goto out;
    }
} else if ( nlh->nlmsg_type < cb_ctl_array_len ) {
    if ( cb_ctl_array && cb_ctl_array[ nlh->nlmsg_type ] ) {
        ret = cb_ctl_array[ nlh->nlmsg_type ]( nlh, data );
        if ( ret <= MNL_CB_STOP )
            goto out;
    }
} else if ( default_cb_array[ nlh->nlmsg_type ] ) {
    ret = default_cb_array[ nlh->nlmsg_type ]( nlh, data );
    if ( ret <= MNL_CB_STOP )
        goto out;
}
```

在这里可以看见，如果响应报文的类型大于等于`NLMSG_MIN_TYPE`，则说明这个响应是数据信息，应当交由`cb_data`这个回调函数进行执行：

<code-block lang="C++">
cb_data = rd_dev_init_cb( const struct nlmsghdr* nlh, void* data );
</code-block>

如果响应报文的类型位于`mnl`规定的类型之内，且我们实现了对应类型的回调，则优先考虑实现的回调:

<code-block lang="C++">
static mnl_cb_t mnlu_cb_array[ NLMSG_MIN_TYPE ] = {
  [NLMSG_NOOP]    = mnlu_cb_noop,
  [NLMSG_ERROR]   = mnlu_cb_error,
  [NLMSG_DONE]    = mnlu_cb_stop,
  [NLMSG_OVERRUN] = mnlu_cb_noop,
};

ret = cb_ctl_array[ nlh->nlmsg_type ]( nlh, data );
</code-block>

如果响应报文的类型既没有超出`mnl`规定的类型范围，又没有实现对应类型的回调，则使用`mnl`的默认实现：

<code-block lang="C++">
static const mnl_cb_t default_cb_array[ NLMSG_MIN_TYPE ] = {
  [NLMSG_NOOP]    = mnl_cb_noop,
  [NLMSG_ERROR]   = mnl_cb_error,
  [NLMSG_DONE]    = mnl_cb_stop,
  [NLMSG_OVERRUN] = mnl_cb_noop,
};

ret = default_cb_array[ nlh->nlmsg_type ]( nlh, data );
</code-block>

**经过实际调试，在初始化时，如果请求被正确响应，则返回的报文类型应该是`NLMSG_DONE`，由于我们内部实现了`mnl_cb_t`，因此会执行`mnlu_cb_stop`**：

``` c++
if ( mnl_nlmsg_get_payload_len( nlh ) < sizeof( len ) )
    return MNL_CB_STOP;
len = *(int*) mnl_nlmsg_get_payload( nlh );
```
