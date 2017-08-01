# RabbitMQ入门指南（6）--远程过程调用

**声明**：本文都是参考RabbitMQ的官方指南翻译过来的，由于本人水平有限难免有翻译不当的地方，敬请谅解。

> **前提条件**

> 本教程假定RabbitMQ 已在标准端口（5672）上的localhost上安装并运行。如果使用不同的主机，端口或凭据，连接设置将需要调整。 

*摘要: 实现远程调用*

## 远程过程调用-RPC（使用Java客户端）

在第二个教程中，我们学习了如何使用工作队列将耗时的任务分配到多个工作者中。

但是如果我们需要在远程计算机上的函数运行功能并等待结果怎么办？那么这就是另外一种模式了，此模式通常称为远程过程调用或RPC。

在本教程中，我们将使用RabbitMQ构建一个RPC系统：一个客户端和一个可扩展的RPC服务器。由于我们没有任何值得分散的耗时任务，我们将会创建一个虚拟的RPC服务，用来返回Fibonacci(斐波纳契数列)。

## 用户接口

为了说明如何使用RPC服务，我们将创建一个简单的客户端类。它将暴露一个名为call的方法，该方法 发送RPC请求，并在响应回复之前都会一直阻塞：

	FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();
	String result = fibonacciRpc.call("4");
	System.out.println( "fib(4) is " + result);

> **关于RPC**

> 虽然RPC是一个很常见的计算模式，但是它依旧常常受批判的。当程序员不知道函数调用是本地函数还是缓慢的RPC时，会出现问题。这样的混乱导致了一个不可预知的系统，并增加了调试的不必要的复杂性。而不是简化软件，滥用RPC可能导致不可维护的意大利面条代码（译者注：原文是spaghetti code可能形容代码很长很乱）。

> 铭记这一点，请考虑以下建议：

> 确保显而易见哪个函数调用是本地的，哪个是远程调用的。

> 给您的系统加上文档，让组件之间的依赖项清晰可见的。

> 处理错误情况。当RPC服务器长时间停机时，客户端应该如何反应？

> 于RPC的所有疑问消除，在你可以的情况下，你应该使用一个异步的管道，代替RPC中阻塞，结果会异步的放入接下来的计算平台。

## 回调队列

一般来说在RabbitMQ上做RPC是容易的。客户端发送请求消息，服务器返回响应消息。为了收到响应，我们需要在请求中带上一个callback队列的地址。我们可以使用默认队列（在Java客户端中是独占的）。让我们试一下：

	callbackQueueName = channel.queueDeclare().getQueue();
	
	BasicProperties props = new BasicProperties
	                            .Builder()
	                            .replyTo(callbackQueueName)
	                            .build();
	
	channel.basicPublish("", "rpc_queue", props, message.getBytes());
	
	// ... then code to read a response message from the callback_queue ...

> **消息属性**

> AMQP协议预先定义了一组消息中的14个属性。大多数属性很少使用，除了下面这些例外:

> deliveryMode：将消息标记为持久化（值为2）或瞬态的（任何其他值）。您可能会从第二个教程中记住此属性。

> contentType：用于描述媒体类型的编码。例如对于经常使用的JSON编码，将此属性设置为：application/json是一个很好的做法。

> replyTo：通常用来命名一个回调队列。

> correlationId：用于将RPC响应与请求相关联。

我们需要这个新的引用：

	import com.rabbitmq.client.AMQP.BasicProperties;

## 相关标识

在上面提出的方法中，我们建议为每个RPC请求创建一个回调队列。这是非常低效的，但幸运的是有一个更好的方法 ，让我们为每个客户端创建一个回调队列。

这样又出现了新的问题，没有清晰的判断队列中的响应是属于哪个请求的。这个时候coorrelationId属性发挥了作用。我们将每个请求的这个属性设置为唯一值。之后，当我们在回调队列中收到一条消息时，我们将查看此属性，并且基于此，我们将能够将响应与请求相匹配。如果我们遇见个未知的correlationId值，我们可以放心地丢弃这个消息，因为它不属于任何一个我们的请求。

您可能会问，为什么我们要忽略哪些在回收队列中未知的消息，而不是以一个错误结束？这是由于在服务器端发生竞争条件下，这种情况是有可能的。RPC服务器发送给我们答应之后，在发送一个确认消息之前，就死掉了，虽然这种可能性不大，但是它依旧存在可能。如果发生这种情况，RPC服务器重启之后，将会再一次处理请求。这就是为什么在客户端上，我们必须优雅地处理这些重复的响应，这RPC理想情况下是幂等的。

## 摘要

![python-six.png](http://www.rabbitmq.com/img/tutorials/python-six.png)

我们的RPC将会像这样工作：

-----------------------------------------

当客户端启动时，它会创建一个匿名的独占的回收队列。

对于一个RPC请求，客户端会发送一个消息中有两个属性：  replyTo，它被设置为回调队列和correlationId对于每一个请求都是唯一值。

这请求发送到rpc_queue队列中。

这RPC工作者(亦称：服务器)正在等待队列上的请求。当请求出现时，它将执行该作业，并使用replyTo字段中的队列将结果发送回客户端。

客户端等待回调队列中的数据。当信息出现时，它会检查correlationId属性。如果它与请求中的值相匹配，则它会返回这响应给应用程序。

-------------------------------------------

## 把它们放在一起

斐波那契任务：

	private static int fib(int n) {
	    if (n == 0) return 0;
	    if (n == 1) return 1;
	    return fib(n-1) + fib(n-2);
	}

我们声明我们的斐波那契函数。它假定一个合法的正整数做为输入参数。（不要期望这个可以处理大量数字，它可能是最慢的递归实现了）。

 我们的RPC服务器[RPCServer.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCServer.java)的代码如下所示：
 
	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	import java.util.concurrent.TimeoutException;
	
	public class RPCServer {
	
	    private static final String RPC_QUEUE_NAME = "rpc_queue";
	
	    public static void main(String[] argv) {
	        ConnectionFactory factory = new ConnectionFactory();
	        factory.setHost("localhost");
	
	        Connection connection = null;
	        try {
	            connection      = factory.newConnection();
	            Channel channel = connection.createChannel();
	
	            channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);
	
	            channel.basicQos(1);
	
	            System.out.println(" [x] Awaiting RPC requests");
	
	            Consumer consumer = new DefaultConsumer(channel) {
	                @Override
	                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
	                    AMQP.BasicProperties replyProps = new AMQP.BasicProperties
	                            .Builder()
	                            .correlationId(properties.getCorrelationId())
	                            .build();
	
	                    String response = "";
	
	                    try {
	                        String message = new String(body,"UTF-8");
	                        int n = Integer.parseInt(message);
	
	                        System.out.println(" [.] fib(" + message + ")");
	                        response += fib(n);
	                    }
	                    catch (RuntimeException e){
	                        System.out.println(" [.] " + e.toString());
	                    }
	                    finally {
	                        channel.basicPublish( "", properties.getReplyTo(), replyProps, response.getBytes("UTF-8"));
	
	                        channel.basicAck(envelope.getDeliveryTag(), false);
	                    }
	                }
	            };
	
	            channel.basicConsume(RPC_QUEUE_NAME, false, consumer);
	
	            //...
	        }
	    }
	}

这服务器代码是相当简单明了的： 

-----------------------------------------------------

如往常一样，我们开始建立连接，通道和声明队列。 我们可能想运行不止一个服务器进程。

为了在多个服务器上平均分配负载，我们需要设置channel.basicQos中的prefetchCount属性。 我们使用basicConsume访问队列，​​以对象（DefaultConsumer）的形式提供一个回调，该对象将执行该操作并发回响应。

-----------------------------------------------------

我们的RPC客户端代码[RPCClient.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCClient.java)的代码：

	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	import java.util.UUID;
	import java.util.concurrent.ArrayBlockingQueue;
	import java.util.concurrent.BlockingQueue;
	import java.util.concurrent.TimeoutException;
	
	public class RPCClient {
	
	    private Connection connection;
	    private Channel channel;
	    private String requestQueueName = "rpc_queue";
	    private String replyQueueName;
	
	    public RPCClient() throws IOException, TimeoutException {
	        ConnectionFactory factory = new ConnectionFactory();
	        factory.setHost("localhost");
	
	        connection = factory.newConnection();
	        channel = connection.createChannel();
	
	        replyQueueName = channel.queueDeclare().getQueue();
	    }
	
	    public String call(String message) throws IOException, InterruptedException {
	        String corrId = UUID.randomUUID().toString();
	
	        AMQP.BasicProperties props = new AMQP.BasicProperties
	                .Builder()
	                .correlationId(corrId)
	                .replyTo(replyQueueName)
	                .build();
	
	        channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));
	
	        final BlockingQueue<String> response = new ArrayBlockingQueue<String>(1);
	
	        channel.basicConsume(replyQueueName, true, new DefaultConsumer(channel) {
	            @Override
	            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
	                if (properties.getCorrelationId().equals(corrId)) {
	                    response.offer(new String(body, "UTF-8"));
	                }
	            }
	        });
	
	        return response.take();
	    }
	
	    public void close() throws IOException {
	        connection.close();
	    }
	
	    //...
	}

这客户端代码是更加清晰：

------------------------------------------------------

我们建立一个连接和通道并且声明一个独占的callback队列用来等待答复。 
 
我们订阅这个callback队列，以便于我们可以接收到RPC响应。
 
我们的call方法做这真正的RPC请求。  

在这里，我们首先生成一个唯一的correlationId 数字并保存它 - 我们 在DefaultConsumer中实现handleDelivery将使用此值来捕获适当的响应。
  
接下来，我们发布请求消息，其中包含两个属性：  replyTo和correlationId。 

这时候，我们可以坐下来，等着合适的响应抵达。 

由于我们的消费者交付处理发生在一个单独的线程中，我们将需要一些东西在响应到达之前暂停主线程。用法的BlockingQueue是可能的解决方案之一。这里我们正在创建ArrayBlockingQueue ，容量设置为1，因为我们只需要等待一个响应。

该handleDelivery方法是做一个很简单的工作，对每一位消费响应消息它会检查的correlationID 是我们要找的人。如果是这样，它将对BlockingQueue的响应。

同时主线程正在等待响应从BlockingQueue中取出。

最后，我们将响应返回给用户。

------------------------------------------------------

客户端请求：

	RPCClient fibonacciRpc = new RPCClient();
	
	System.out.println(" [x] Requesting fib(30)");
	String response = fibonacciRpc.call("30");
	System.out.println(" [.] Got '" + response + "'");
	
	fibonacciRpc.close();

现在是时候让我们回顾下我们RPCClient.java和RPCServer.java中的全部例子的源码(包含基本的异常处理)。 

像往常一样编译和设置类路径（参见教程一）：

	javac -cp $CP RPCClient.java RPCServer.java

我们的RPC服务现在已经准备就绪。我们可以启动服务器：

	java -cp $CP RPCServer
	# => [x] Awaiting RPC requests

为了请求一个斐波那契数字，运行客户端：

	java -cp $CP RPCClient
	# => [x] Requesting fib(30)

这里提出的设计不仅仅可以实现一个RPC服务，并且它还有几项重要的优势：

如果RPC服务器反应太迟缓，可以通过运行另一个RPC服务器进行扩展。尝试在新的控制台中运行第二个RPCServer。

在客户端，RPC需要发送和接收一条消息。不需要同步调用，如queueDeclare 。因此，RPC客户端只需要一个网络往返单个RPC请求。

我们的代码仍然非常简单，不能解决更复杂（但重要的）问题，如：

- 如果没有服务器运行，客户端应该如何反应？

- 客户端是否需要RPC的超时有处理？

- 如果服务器发生故障，抛出一个异常，是否应该传递到客户端？

- 在处理之前防止无效的传入消息（例如检查边界，类型）

*如果你想实验下，你会发现rabbitmq-management插件对观察队列是很有帮助的。*





