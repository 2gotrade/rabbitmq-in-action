# 队列长度限制

## 介绍

队列的最大长度可以限制消息的数量或字节数（所有消息体长度的总和，忽略消息属性和任何开销），或两者兼而有之。

对于任何给定的队列，可以由队列的客户端同参数或在服务器中使用策略定义最大长度（任一类型）。在策略和参数指定最大长度的情况下，选用两个值中的最小值。

在任何情况下，已经被使用但未确认的消息不会计入限制。来自  rabbitmqctl list_queues的messages_ready和message_bytes_ready字段以及 管理API显示是有限制的。 

一旦达到限制，消息将从队列的前面丢弃或者死亡，为新消息留出空间。

## 使用策略定义最大队列长度

要使用策略指定最大长度，需要将关键字max-length和（或）max-length-bytes添加 到策略定义中。例如：

**rabbitmqctl**	

	rabbitmqctl set_policy Ten "^one-meg$" '{"max-length-bytes":1000000}' --apply-to queues

**rabbitmqctl (Windows)**	

	rabbitmqctl.bat set_policy Ten "^one-meg$" "{""max-length-bytes"":1000000}" --apply-to queues

这样可以确保名称为one-meg的队列可以包含不超过1MB的消息体。

还可以使用管理插件定义[策略](http://www.rabbitmq.com/parameters.html#policies)，有关详细信息，请参阅[策略文档](http://www.rabbitmq.com/parameters.html#policies)。

## 在声明期间使用x参数定义最大队列长度

可以通过向非负整数值提供x-max-length队列声明参数来设置最大消息 数。

可以通过以非负整数值提供x-max-length-bytes队列声明参数来设置最大长度（以字节  为单位）。

如果两个参数都被设置，则两个参数都可以应用; 首先要限制的限制将被强制执行。

以下的Java示例声明一个队列的最大长度为10个消息：

	Map<String, Object> args = new HashMap<String, Object>();
	args.put("x-max-length", 10);
	channel.queueDeclare("myqueue", false, false, false, args);
