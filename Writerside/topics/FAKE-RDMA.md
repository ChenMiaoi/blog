# FAKE RDMA

为了快速学习RDMA协议以及内部核心原理，做此记录。

## First Week

> Target  
> 1. 完成一个fake-rdma device(uverbs)构建，能够在/dev/下发现该设备(kernel layer，refer erdma)
> 2. 探究ibv_device(ibv_devinfo)是否与provider直接/间接关联，并能够成功访问我们创建的设备

