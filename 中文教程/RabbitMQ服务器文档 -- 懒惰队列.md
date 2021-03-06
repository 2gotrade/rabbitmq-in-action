# 懒惰队列

从RabbitMQ 3.6.0版本开始，broker引入了延迟队列的概念：这些队列可以尽可能多地保留在磁盘上的消息，并且只有在消费者请求时才将它们加载到RAM中，因此这样的队列称为懒惰队列。

懒惰队列的主要目的是为了能够支持很长的队列（数百万条消息）。当消费者无法长时间地从队列中获取消息时，就可能会产生这样的队列。它之所以会存在，是因为各种原因和用例：消费者是离线的；它们已经崩溃了，或者它们已经被删除；等等。

默认情况下，当消息发布到RabbitMQ中时，队列的消息被存放于内存中缓存。这种缓存的思想能够尽可能快地向消费者传递消息（请注意，持久消息在进入broker并在同一时间保存在此缓存中时被写入磁盘）。每当broker认为需要释放内存时，此缓存中的消息将被调出到磁盘。将消息传递到磁盘需要时间并阻塞队列进程，使其在分页时无法接收新消息。即使在最近的RabbitMQ版本我们改进了分页算法，在某些情况下仍不理想：你有数百万人使用队列中可能需要分页信息。

懒惰队列通过消除此缓存并在消费者请求时仅在内存中加载消息来帮助解决上述的问题。懒惰队列会将每个到达队列的消息立即发送到文件系统，从而完全消除前面提到的内存中缓存。这就大大地减少了队列消耗的RAM，并且也消除了分页的消耗。虽然这将增加I / O使用率，但它与发布持久性消息时的行为相同。

## 使用懒惰队列

通过使用queue.declare参数指定模式，或者通过在服务器中应用策略， 可以将队列设置为默认模式或 延迟（懒惰）模式 。在策略和队列参数均指定队列模式的情况下，队列参数的优先级高于策略值。这意味着如果通过declare参数设置队列模式，则只能通过删除队列来更改队列模式，然后再使用不同的参数重新声明。 

### 使用参数进行配置

可以通过向指定所需模式的字符串提供x队列模式队列声明参数来设置 队列模式。有效模式是  “默认”和“懒惰”。如果在声明期间未指定模式，则默认为“default”。在默认模式是行为已经存在于3.6.0之前版本的broker，所以在这方面没有重大更改。

如下示例，在Java中声明队列模式设置为“lazy”的队列：

	Map<String, Object> args = new HashMap<String, Object>();
	args.put("x-queue-mode", "lazy");
	channel.queueDeclare("myqueue", false, false, false, args);

### 使用策略配置

如果要使用策略指定队列模式，请将队列长度关键字添加 到策略定义中。例如：	
	
**rabbitmqctl**	

	rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"lazy"}' --apply-to queues

**rabbitmqctl (Windows)**
	
	rabbitmqctl set_policy Lazy "^lazy-queue$" "{""queue-mode"":""lazy""}" --apply-to queues	
	
这样可以确保延迟队列的队列将以懒惰模式工作。

还可以使用管理插件定义策略，有关详细信息，请参阅[策略文档](http://next.rabbitmq.com/parameters.html#policies)。	

### 更改队列模式

如果您通过策略指定队列模式，则可以在运行时更改队列模式，而无需删除队列，并以不同的模式重新声明。如果您希望先前的延迟队列开始像默认队列一样工作 ，那么可以通过发出以下命令来执行此操作：

**rabbitmqctl**
	
	rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"default"}' --apply-to queues

**rabbitmqctl (Windows)**
	
	rabbitmqctl set_policy Lazy "^lazy-queue$" "{""queue-mode"":""default""}" --apply-to queues

## 懒惰队列的性能考虑

### 磁盘利用率

如上所述，懒惰队列将在进入队列时将每个消息发送到磁盘。与持续发送消息的“常规”持久队列相比，这将增加磁盘I/O利用率。请注意，即使 是使用懒惰队列的瞬态消息仍将发送到磁盘。使用 默认队列，如果需要分页，临时消息只会发送到磁盘。 

### RAM利用率

懒惰队列使用比默认队列少得多的内存 。虽然很难给出每个用例都有意义的数字，但是这有一个比较对比的例子：发布1000万条消息路由到队列中，而没有任何消费者在线。消息正文大小为1000字节。默认队列模式需要1.2GB的RAM，而懒惰队列只使用1.5MB的RAM。

对于默认队列，发送1000万条消息需要801秒，平均发送速率为12469 msg/s。要将相同数量的消息发布到一个懒惰 队列中，所需时间为421秒，平均发送速率为23653 msg/s。这样的差异可以通过以下事实来解释：事实上，默认 队列不得不将消息寻址到磁盘。一旦我们激活了一个消费者，懒惰的队列在传递消息的时候就消耗了大约40MB的RAM。一个活跃消费者的消息接收率平均为13938 msg/s。

您可以通过运行我们的Java库来重现测试：

	./runjava.sh com.rabbitmq.examples.PerfTest -e test -u test_queue \
	  -f persistent -s 1000 -x1 -y0 -C10000000

请注意，这是一个非常简单的测试。请确保运行您自己的基本条件。

不要忘记在基本条件运行之间更改队列模式。

### 在队列模式之间转换

如果我们需要将一个默认队列转换成一个懒惰的队列 ，那么我们将受到同样的性能影响，当队列需要将消息传递到磁盘时。当我们将一个队列转换成一个懒惰的队列时，首先将消息传递到磁盘，然后它将开始接受发布，acks和其他命令。

当队列从懒惰模式迁移到默认模式时 ，它将执行与在服务器重新启动后恢复队列时相同的进程。在上述缓存中将加载一批16384条消息。


	