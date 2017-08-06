# 消费者取消通知

当一个通道在队列中消费消息时，有各种原因可能导致消费停止。其中一个就是如果客户端在同一个通道上发出一个  basic.cancel，这将导致消费者被取消，并且服务器使用basic.cancel-ok进行回复。其他事件，例如队列被删除，或者在集群场景中，队列所在的节点出现故障，也将导致消费被取消，但不会通知客户端信道，这通常是无益的。

为了解决这个问题，我们引入了一个扩展，在 这种消费者被意外取消的情况下，broker将向客户发送一个basic.cancel。在broker从客户端接收basic.cancel的情况下不发送 。默认情况下，AMQP 0-9-1客户端不希望从broker异步接收  basic.cancel方法，因此为了实现此行为，客户端必须在其客户端属性中提供一个功能表，其中有一个关键consumer_cancel_notify和boolean值呈现能力表。 

我们支持的客户端默认地向broker提供此功能，因此将由broker发送异步  basic.cancel方法，broker程序将它们发送给消费者回调。例如，在我们的Java客户端中，Consumer接口有一个  handleCancel回调，可以通过对DefaultConsumer类进行子类化来覆盖：

	channel.queueDeclare(queue, false, true, false, null);
	Consumer consumer = new DefaultConsumer(channel) {
	    @Override
	    public void handleCancel(String consumerTag) throws IOException {
	        // consumer has been cancelled unexpectedly
	    }
	};
	channel.basicConsume(queue, consumer);	
	
对于已经意外取消的消费者（例如由于排队删除） ，客户端发出basic.cancel不会发生错误 。根据定义，在发出basic.cancel的客户端和发送异步通知的broker之间存在竞争。在这种情况下，当接收到basic.cancel并且以正常的basic.cancel-ok进行回复 时，broker不会出错 。

## 消费者取消和HA故障转移	

当队列被删除或不可用时，将始终通知支持消费者取消通知的客户端。消费者可能会要求在镜像队列故障转移时取消它们（请参阅镜像队列中的页面为什么以及如何完成此操作）。

### 客户端和服务器功能

AMQP 0-8规范定义了一个 功能字段作为connection.open方法的一部分 。该字段在AMQP 0-9-1规范中已被废弃，不被RabbitMQ检查。如AMQP 0-8所述，它也是一个shortstr：一个不超过256个字符的字符串。

为了客户端和服务器都有能力提供他们支持的扩展和功能，我们引入了一种替代形式的功能。在connection.start的服务器属性字段中和在connection.start-ok的  客户端属性字段中 ，字段值（  对等体属性表）可以可选地包含一个名为capabilities的键，其值为另一个表，其中的键命名支持的功能。这些功能键的值通常是布尔值， 

例如，RabbitMQ代理向客户端呈现的服务器属性可能如下所示：

	{ "product"      = (longstr) "RabbitMQ",
	  "platform"     = (longstr) "Erlang/OTP",
	  "information"  = (longstr) "Licensed under the MPL.  See http://www.rabbitmq.com/",
	  "copyright"    = (longstr) "Copyright (C) 2007-2016 Pivotal Software, Inc.",
	  "capabilities" = (table)   { "exchange_exchange_bindings" = (bool) true,
	                               "consumer_cancel_notify"     = (bool) true,
	                               "basic.nack"                 = (bool) true,
	                               "publisher_confirms"         = (bool) true },
	  "version"      = (longstr) "2.4.0" }	

请注意，客户端可以将此功能表 呈现为客户 端属性表的一部分：未能呈现此类表不排除客户端能够使用扩展（如交换来交换绑定）。然而，在某些情况下，例如消费者取消通知，客户端必须呈现相关联的能力，否则代理人将无法知道客户端能够接收附加通知。


	