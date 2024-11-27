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
        init dev_map_list   :a1, 0, 10s
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

