# 消费者预取

一个限制未确认消息的更自然和有效的方式。

AMQP 0-9-1指定basic.qos方法，允许您 在消费（也称为“预取计数”）时限制通道（或连接）上未确认消息的数量。不幸的是，信道通常不是理想的范围 ，因为单个通道可能会从多个队列中消耗，通道和队列需要为每个发送的消息进行协调，以确保它们不超过极限。这在一台机器上速度很慢，在集群中消耗很慢。

此外，对于许多应用程序来说，指定适用于每个消费者的预取计数更为自然。

因此，RabbitMQ 在basic.qos方法中重新定义了 全局（global）标志的含义：

**AMQP 0-9-1 中prefetch_count的含义**

- false - 在通道上的所有消费者共享
- true - 共享所有消费者的连接

**RabbitMQ 中prefetch_count的含义**

- false - 分别应用于通道上的每个新消费者
- true - 在通道上的所有消费者共享

请注意，在大多数API中，全局标志的默认值为false。

## 示例 - 单个消费者

以下基于Java示例将一次最多可以收到10个未确认的消息：

	Channel channel = ...;
	Consumer consumer = ...;
	channel.basicQos(10); // Per consumer limit
	channel.basicConsume("my-queue", false, consumer);

## 示例 - 多个独立消费者

此示例在同一通道上启动两个消费者，每个用户将一次最多可独立接收10个未确认的消息：

	Channel channel = ...;
	Consumer consumer1 = ...;
	Consumer consumer2 = ...;
	channel.basicQos(10); // Per consumer limit
	channel.basicConsume("my-queue1", false, consumer1);
	channel.basicConsume("my-queue2", false, consumer2);

## 示例 - 多个消费者共享限制

AMQP规范没有解释如果使用不同的全局值多次 调用basic.qos会发生什么。RabbitMQ将此解释为意味着两个预取限制应该彼此独立地执行; 消费者只有在没有达到未确认消息的限制时才会收到新消息。 

例如：

	Channel channel = ...;
	Consumer consumer1 = ...;
	Consumer consumer2 = ...;
	channel.basicQos(10, false); // Per consumer limit
	channel.basicQos(15, true);  // Per channel limit
	channel.basicConsume("my-queue1", false, consumer1);
	channel.basicConsume("my-queue2", false, consumer2);

这两个消费者在他们之间只会有15个未确认的消息，每个消费者最多可以有10个消息。这将比上述示例慢，这是由于在通道和队列之间进行协调以增加全局限制的额外开销。	