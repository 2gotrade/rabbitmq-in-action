# Net Tick Time（网络时间）

本指南介绍了RabbitMQ节点和CLI工具（以及Erlang节点）用于确定对等体被称为“net ticks”或  kernel.net_ticktime机制的可用性。有关更多详细信息，请参阅[Erlang内核文档](http://erlang.org/doc/man/kernel_app.html)。

## 概述

集群中的每对节点之间由传输层连接。在所有节点对之间定期交换消息，以保持连接并检测连接是否断开。否则网络中断可能会在相当长的一段时间内被检测到（取决于传输和操作系统内核设置，例如TCP）。从根本上说这是心跳寻求在通讯协议解决同样的问题，只是在不同的对等体之间：RabbitMQ集群节点和CLI工具。

节点和连接的CLI工具周期性地发送对方的小数据帧。如果在给定时间内没有从对等体接收到数据，则该对等体被认为是不可访问的（“down”）。

当一个RabbitMQ节点确定另一个节点已经关闭时，它将记录一条消息，给出另一个节点的名称和原因，如：

	=INFO REPORT==== 23-Sep-2014::16:21:22 ===
	node rabbit@cordelia down: net_tick_timeout

在这种情况下，net_tick_timeout告诉我们，由于超过了net ticktime，其他节点被检测为down。另一个常见的原因是  connection_closed，这意味着连接在TCP级别被显式关闭。

## 瞬间频率（Tick Frequency）

通过net_ticktime配置控制两个tick消息和故障检测的频率。通常在每个net_ticktime秒之间的一对节点之间交换四个tick 。如果在net_ticktime（±25％）秒内没有从节点接收到消息，则该节点被认为已关闭，不再是集群的成员。

在集群中的所有节点上 增加 net_ticktime将使集群对短网络的出口更有弹性，但是重新节点需要更长时间才能检测到崩溃的节点。相反，减少集群中所有节点的 net_ticktime会降低检测延迟，但会增加检测虚假分区的风险 。

应该慎重考虑更改默认net_ticktime的影响。集群中的所有节点必须使用相同的  net_ticktime。以下示例rabbitmq.config 配置显示将默认net_ticktime从60秒增加到120秒：

    [
        {rabbit, [{tcp_listeners, [5672]}]},
        {kernel, [{net_ticktime,  120}]}
    ].

## HTTP API

HTTP API常常需要执行集群范围的查询，这将导致UI在检测和处理一个分区之前无响应显示。降低net_ticktime 可以帮助提高种情况的响应速度，但是正如上文所强调的，任何改变net_ticktime的决定都应该慎重考虑。

