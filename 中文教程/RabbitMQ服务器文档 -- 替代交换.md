# 替代交换

有时我们希望让客户端处理交换机无法路由的消息（即没有绑定的队列，我们​​没有匹配的绑定）。这方面的典型例子是：

- 检测客户端是否意外或恶意发布无法路由的邮件
- 某些消息特别处理的路由语义，其余的由通用处理程序处理

RabbitMQ的替代交换（“AE”）功能解决了这些用例。

对于任何给定的交换，AE可以由客户端使用交换机的参数来定义，或者在使用策略的服务器中定义。在策略和参数指定AE的情况下，参数中指定的值会覆盖策略中指定的值。

## 使用参数进行配置

创建交换机时，可以通过指定“alternate-exchange”的键和包含该名称的“S”（字符串）值，在exchange.declare 方法的参数表中选择提供AE 的名称。

当指定AE时，除了声明的交换机上的常规 配置权限之外，用户需要对该交换机具有读取权限并对AE 写入权限。

例如：

	Map<String, Object> args = new HashMap<String, Object>();
	args.put("alternate-exchange", "my-ae");
	channel.exchangeDeclare("my-direct", "direct", false, false, args);
	channel.exchangeDeclare("my-ae", "fanout");
	channel.queueDeclare("routed");
	channel.queueBind("routed", "my-direct", "key1");
	channel.queueDeclare("unrouted");
	channel.queueBind("unrouted", "my-ae", "");

在上面的Java代码片段中，我们创建了一个直接交换“my-direct”，配置了一个名称为“my-ae”的AE。后者被声明为fanout类型的交换。我们使用'key1'的绑定关键字绑定一个'routed'到'my-direct'队列，并将队列'unrouted'绑定到'my-ae'。

## 使用策略配置

要使用策略指定AE，请将关键字“alternate-exchange”添加到策略定义。例如：

**rabbitmqctl**
	
	rabbitmqctl set_policy AE "^my-direct$" '{"alternate-exchange":"my-ae"}'

**rabbitmqctl (Windows)**
	
	rabbitmqctl set_policy AE "^my-direct$" "{""alternate-exchange"":""my-ae""}"

这时我们将“my-ae”的AE称为“my-direct”交换。还可以使用管理插件定义策略，有关详细信息，请参阅策略文档。

## 备用交换逻辑

每当与配置的AE的交换无法将消息路由到任何队列时，它会将消息发布到指定的AE。如果AE不存在，则会记录一个警告。如果AE不能路由消息，它会将消息发布到其AE，如果它已经配置了一个消息。该过程一直持续到消息成功路由，达到AE链的结束，或者遇到已经尝试路由消息的AE。

例如，如果我们使用'key1'的路由关键字向'my-direct'发布消息，则该消息将根据标准AMQP行为路由到“routed”队列。然而，当使用'key2'的路由关键字发布消息到'my-direct'时，消息将通过我们配置的AE路由到'unrouted'队列而不是被丢弃。

AE的行为完全属于路由。如果消息通过AE路由，它仍然被计为路由以作为“强制”标志的目的，并且消息以其他方式保持不变。


	