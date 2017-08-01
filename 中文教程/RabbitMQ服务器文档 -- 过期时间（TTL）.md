# TTL，Time-To-Live Extensions（过期时间）

RabbitMQ允许您为消息（message ）和队列（queue ）设置TTL（生存时间）。这可以使用可选的队列参数或 策略完成（推荐使用后一个选项）。可以为单个队列，一组队列或单个消息应用消息TTL。

## 队列中每个消息的过期时间（TTL）

通过在 queue.declare 中设置 x-message-ttl 参数，可以控制被 publish 到 queue 中的 message 被丢弃前能够存活的时间。当某个 message 在 queue 留存的时间超过了配置的 TTL 值时，我们说该 message “已死”。值得注意的是，当一个 message 被路由到多个 queue 中时，其可以在不同的时间“死掉”，或者可能有的不会出现“死掉”情况。消息在一个队列中的死亡对其他队列中同一个消息的生命没有影响。

服务器保证不会使用basic.deliver（对消费者）或包含在basic.get-ok响应（用于一次性获取操作）中传递“已死”信息。此外，服务器将尝试在其基于TTL的到期后或不久之后收到消息。

参数 x-message-ttl 的值 必须是非负 32 位整数 (0 <= n <= 2^32-1) ，以毫秒为单位表示 TTL 的值。这样，值 1000 表示存在于 queue 中的当前 message 将最多只存活 1 秒钟，除非其被投递到 consumer 上。实参可以是以下 AMQP 类型：short-short-int 、short-int 、long-int 或者 long-long-int 。 

**使用策略定义队列的消息TTL**

为了使用策略指定TTL，需要将关键字“message-ttl”添加到策略定义中：

---------------------------------------------------------------------------------------------------------

**rabbitmqctl**

	rabbitmqctl set_policy TTL ".*"'{"message-ttl":60000}'--apply-to queues

**rabbitmqctl (Windows)**

	rabbitmqctl set_policy TTL ".*" "{""message-ttl"":60000}" --apply-to queues

---------------------------------------------------------------------------------------------------------

这样对所有队列应用了60秒的TTL。

**在声明期间使用参数定义队列的消息TTL**

下面的 Java 示例创建一个队列，其中消息最多能存活 60 秒： 

	Map<String, Object> args = new HashMap<String, Object>();
	args.put("x-message-ttl", 60000);
	channel.queueDeclare("myqueue", false, false, false, args);

使用.NET客户端的同一个例子：

	var args = new Dictionary<string, object>();
	args.Add("x-message-ttl", 60000);
	model.QueueDeclare("myqueue", false, false, false, args);

可以对已经有消息的队列应用消息TTL策略，但这涉及到[一些注意事项](http://www.rabbitmq.com/ttl.html#per-message-ttl-caveats)

当消息被重新排序（requeue）的时候，其原始过期时间将被保留（例如由于设置了 requeue 参数的 AMQP 方法的使用，或者由于 channel 的关闭）。

将TTL设置为0会导致消息在到达队列时（尚未被立即投递到消费者）判定为过期，除非它们可以立即发送给消费者。因此，这种方式相当于  basic.publish中立即（immediate）标识情况下的等价实现，RabbitMQ服务器不支持。与该标志不同，将不会有 basic.returns 命令的调用，但是在设置了 dead letter exchange 的情况下，这些 message 将被处理为 dead-lettered（详见下面的DLX）。

## 发布者中每个消息的TTL

通过 basic.publish 命令发送消息 时设置 expiration字段， 可以在每个消息的基础上指定TTL。

expiration 字段以以毫秒为单位表示 TTL 值。且与 x-message-ttl 具有相同的约束条件。因为 expiration 字段必须为字符串类型，因此broker将只会接受以字符串形式表达的数字。 

当指定了每个队列（queue）和每个消息（message）的TTL时，将选择两者之间的较低值。

下面的 Java 示例 发布了可驻留在队列中最多60秒的消息：

	byte[] messageBodyBytes = "Hello, world!".getBytes();
	AMQP.BasicProperties properties = new AMQP.BasicProperties();
	properties.setExpiration("60000");
	channel.basicPublish("my-exchange", "routing-key", properties, messageBodyBytes);

.NET中的同一个例子:

	byte[] messageBodyBytes = System.Text.Encoding.UTF8.GetBytes("Hello, world!");
	
	IBasicProperties props = model.CreateBasicProperties();
	props.ContentType = "text/plain";
	props.DeliveryMode = 2;
	props.Expiration = "36000000"
	
	model.BasicPublish(exchangeName,
	                   routingKey, props,
	                   messageBodyBytes);

**注意事项**

虽然 consumer 从来看不到过期的 message ，但是在过期 message 到达 queue 的头部时确实会被真正的丢弃（或者 dead-lettered ）。当对每一个 queue 设置了 TTL 值时不会产生任何问题，因为过期的 message 总是会出现在 queue 的头部。请记住，在消息到期和消费者交付之间可能存在自然竞争条件，例如，消息可以在写入套接字之后但在到达消费者之前过期。

当对每一条 message 设置了 TTL 时，过期的 message 可能会排队于未过期 message 的后面，直到这些消息被 consumer 到或者过期了。因此，这种过期消息使用的资源将不会被释放，并且它们将被计入队列统计信息（例如队列中的消息数）。

当追溯应用每消息TTL策略时，建议让消费者在线，以确保消息更快地被丢弃。

考虑到现有队列中每消息TTL设置的这种行为，当需要删除消息以释放资源时，应使用队列TTL（或排队清除或队列删除）。

## 队列TTL

可以通过将x-expires参数设置为queue.declare或通过设置expires 策略为给定队列设置到期时间 。这将控制队列在被自动删除之前可以使用多长时间。未使用意味着队列没有消费者，队列未被重新声明，并且basic.get至少在过期期限内尚未被调用。这可以用于例如RPC风格的应答队列，其中可以创建可能永远不会被排除的许多队列。

服务器会确保在过期时间到达后 queue 被删除，至少在到期期间不被使用。但是不保证删除的动作有多么的及时。服务器重新启动时，持久队列的超时时间将重新计算。

用于表示超期时间的 x-expires 参数值以毫秒来描述有效期限。它必须是一个正整数（不同于消息TTL，它不能为0）。因此，值为1000表示1秒钟未使用的队列将被删除。

**使用策略为队列定义队列TTL**

以下策略使所有队列在上次使用后30分钟后到期：

--------------------------------

**rabbitmqctl**

	rabbitmqctl set_policy expiry ".*" '{"expires":1800000}' --apply-to queues

**rabbitmqctl (Windows)**

	rabbitmqctl.bat set_policy expiry ".*" "{""expires"":1800000}" --apply-to queues

--------------------------------

**在声明期间使用参数定义队列的队列TTL**

下面的 Java 示例创建了一个 queue ，其会在 30 分钟不使用的情况下判定为超时

	Map<String, Object> args = new HashMap<String, Object>();
	args.put("x-expires", 1800000);
	channel.queueDeclare("myqueue", false, false, false, args);

