# Detecting Dead TCP Connections with Heartbeats（用心跳检测TCP连接）

## 介绍

网络可能会以许多方式连接失败，有时是非常敏感的（例如高比率数据包丢失）。TCP连接超时中断需要较长的时间（例如在Linux上默认配置大约11分钟）才能被操作系统检测到。AMQP提供了心跳功能，以确保应用程序能及时发现连接中断（以及完全无响应的对等体）。心跳也可以防止某些网络设备一段时间内没有任何活动时，可能会终止“空闲”TCP连接。

## 心跳超时间隔

心跳超时值定义在RabbitMQ和客户端库之间，它用来检测同伴的TCP连接在这段时间内是否被视为无法访问（down）的。此值在连接时在客户端和RabbitMQ服务器之间协商。客户端必须配置为请求心跳。在RabbitMQ 3.0及更高版本中，broker将默认尝试协商心跳（尽管客户端仍可否决）。超时时间为秒，默认值为60 （版本3.5.5之前为580）。

每timeout / 2 秒钟发送心跳帧。在两次错过心跳之后，对等体将被认为是无法访问的。不同的客户端显示不同，但是TCP连接将被关闭。当客户端检测到由于心跳而无法访问RabbitMQ节点时，需要重新连接。

任何流量（例如协议操作，发布的消息，确认）都被当成有效的心跳。客户可以选择发送心跳帧，而不管连接是否有其他流量，但有些只能在必要时进行。

可以通过将超时间隔设置为0来禁用心跳。这样的做法不推荐。

**使用Java客户端启用心跳**

要在Java客户机中配置心跳超时，就需要在创建连接之前使用ConnectionFactory＃setRequestedHeartbeat进行设置 ：

	ConnectionFactory cf = new ConnectionFactory();
	
	// set the heartbeat timeout to 60 seconds
	cf.setRequestedHeartbeat(60);

请注意，如果RabbitMQ服务器配置了非零心跳超时（在3.6.x版本中为默认值），则客户端只能降低值但不能增加。

**使用.NET客户端启用心跳**

要在.NET客户端中配置心跳超时，就需要在创建连接之前使用ConnectionFactory.RequestedHeartbeat进行设置 ：

	var cf = new ConnectionFactory();
	
	// set the heartbeat timeout to 60 seconds
	cf.RequestedHeartbeat = 60;

## 低超时值和误报

由于瞬时网络拥塞，短时间内服务器流量控制等原因，将心跳超时值设置得太低可能导致错误的响应（对等体被认为不可用，而不是真的这样）。在选择超时值时应考虑这一点。

来自用户和客户端库维护者的几年价值反馈表明，低于5秒的值很可能会导致误报，1秒或更低的值跟容易导致误报。5到20秒范围内的值对于大多数环境来说是最佳的选择。

## STOMP的心跳

STOMP 1.2包括心跳。在STOMP中，心跳超时可以是不对称的：也就是说，客户端和服务器可以使用不同的值。RabbitMQ STOMP插件完全支持此功能。

STOMP中的心跳选择加入。要启用它们，请 在连接时使用心跳标题。参见STOMP规范的一个例子。

## MQTT的心跳

MQTT 3.1.1包含不同名称的心跳（“keepalives”）。RabbitMQ MQTT插件完全支持此功能。

MQTT中的Keepalives选择加入。要启用它们，请在连接时设置  Keepalive间隔。请参考MQTT客户端文档以获取示例

## Shovel和Federation插件中的心跳

Shovel(http://www.rabbitmq.com/shovel.html)和[Federation](http://www.rabbitmq.com/federation.html)插件打开Erlang客户端连接到RabbitMQ节点下。因此，它们可以被配置为使用期望的心跳值。

有关详细信息， 请参阅[AMQP 0-9-1](http://www.rabbitmq.com/uri-query-parameters.html)查询参数参考。

## 心跳和TCP Keepalives

TCP包含一个与心跳类似的机制（也称为keepalive），是一个在消息传递协议和上面覆盖的net tick超时：TCP keepalives。由于默认值不足，对于消息传递协议，TCP keepalive是不合适的，甚至适得其反。然而，通过适当的调整，在不能通过应用程序启用心跳或使用合理值的环境中，它可以作为额外的防御机制。有关详情，请参阅[网络指南](http://www.rabbitmq.com/networking.html)。

某些网络工具（HAproxy，AWS ELB）和设备（硬件负载均衡器），当一段时间内没有任何活动时，可能会终止“空闲”的TCP连接。大多数时候这是不可取的。

当连接启用心跳时，会导致周期性的轻网络流量。因此，心跳具有保护客户端连接的副作用，该连接可以在一段时间内空闲以防代理和负载均衡器过早关闭。

超过10到30秒的心跳超时将会使网络流量周期性足够（大概每5到15秒））满足绝大多数代理工具和负载均衡器的默认值。另请参阅上述低超时和误报的部分。

