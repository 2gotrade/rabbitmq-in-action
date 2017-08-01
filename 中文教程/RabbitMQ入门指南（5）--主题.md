# RabbitMQ入门指南（5）--主题

**声明**：本文都是参考RabbitMQ的官方指南翻译过来的，由于本人水平有限难免有翻译不当的地方，敬请谅解。

> **前提条件**

> 本教程假定RabbitMQ 已在标准端口（5672）上的localhost上安装并运行。如果使用不同的主机，端口或凭据，连接设置将需要调整。 

*摘要: Publish/Subscribe，根据表达式来接收消息*

## 主题（使用Java客户端）

在上一个教程中，我们改进了我们的日志系统， 而不是使用只能进行虚拟广播的fanout交换器，我们使用一个direct类型的交换器，使有可能选择性地接收日志。

虽然使用direct类型的交换器改进了我们的系统，但它仍然有限制，不能根据多个条件进行路由选择。

在我们的日志记录系统中，我们可能不仅要根据严重性订阅日志，还想要基于发出日志的源进行订阅。您可能可能了解到syslog unix tool的概念，那个基于严重性(info/warn/crit...)和设备(auth/cron/kern...)路由日志。。

这将给我们带来很大的灵活性，我们可能仅仅想监听来自cron的关键性的错误，也可以监听来自kern的所有日志。

要在我们的日志系统中实现这一点，我们需要了解一个更复杂的topic类型交换器。

## 主题交换器

发送到topic类型的交换器不能有任意的路由的关键字，它必须是一个关键字列表，由点分隔。这关键字可以是任意的，但是通常可以说明消息的基本的联系。几个合法的路由关键字例子："stock.usd.nyse"，"nyse.vmw","quick.orange.rabbit"。可能有很多你想要的路由关键字，上限是255个字节。 

这个绑定关键字必须也在这同样的表单里。topic交换器逻辑背后是与direct交换器类型类似，一个带特别的路由关键字的消息将会被传递到所有匹配绑定的关键字的队列。但是绑定关键字两个重要的特殊情况：

1. *（星）可以替代一个字。
2. ＃（哈希）可以替换零个或多个单词。

这个例子中是很容易解释的：

![python-five.png](http://www.rabbitmq.com/img/tutorials/python-five.png)

在这个例子中，我们发送的消息都描述的是动物。被发送的消息的路由关键字是由三个单词（两个点）组成。路由关键字中第一个单词描述的是速度，第二个描述的是颜色，第三个是物种： "<速度>.<颜色>.<物种>"。

我们创建了三个绑定：Q1s是由绑定关键字"*.orange.*"所约束，Q2由"*.*.rabbit"和"lazy.#"所约束。 这些绑定可以概括为：

1. Q1 是对orange颜色的动物感兴趣
2. Q2 想了解关于兔子的所有信息和所有慢吞吞的动物信息

一个路由关键字为"quick.orange.rabbit"消息将会被传递到所有队列。消息"lazy.orange.elephant"同样也传递到所有队列。另一方面"quick.orange.fox"仅进入第一队列，"lazy.brown.fox"仅进入第二个队列。"lazy.pink.rabbit"仅传递到第二个队列一次，即使它会匹配两个绑定。"quick.brown.fox"不符合任何绑定关键字，所以会被丢弃。

如果我们打破我们的约定，发送一个消息带一个或四个单词的关键字，像"orange"或"quick.orange.male.rabbit"，会发生什么呢?那么这些消息将不会匹配任何绑定，并将丢失。

 另一方面"lazy.orange.male.rabbit"，即使它有四个单词，将会匹配最后那个绑定，并将被传递到第二个队列。

> **主题交换器**

> topic类型交换器是强大的，能像其他的交换器一样行事。

> 当一个队列绑定到了"#"(哈希)绑定关键字，它将接收所有消息，而不管路由关键字是什么。

> 当topic类型交换器中没有使用像"*"（星标）和"#"(哈希)的特殊字符，它的行为类似于direct类型交换器。

## 把它们放在一起

我们将会在我们的日志系统中使用topic类型交换器。我们假设我们的工作的日志消息的路由关键字是由两个单词组成，格式为："<facility>.<severity>"。 

代码与上一个教程几乎相同。

[EmitLogTopic.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLogTopic.java)的代码：

	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	
	public class EmitLogTopic {
	
	    private static final String EXCHANGE_NAME = "topic_logs";
	
	    public static void main(String[] argv)
	                  throws Exception {
	
	        ConnectionFactory factory = new ConnectionFactory();
	        factory.setHost("localhost");
	        Connection connection = factory.newConnection();
	        Channel channel = connection.createChannel();
	
	        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
	
	        String routingKey = getRouting(argv);
	        String message = getMessage(argv);
	
	        channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());
	        System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");
	
	        connection.close();
	    }
	    //...
	}

[ReceiveLogsTopic.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogsTopic.java)的代码：

	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	
	public class ReceiveLogsTopic {
	  private static final String EXCHANGE_NAME = "topic_logs";
	
	  public static void main(String[] argv) throws Exception {
	    ConnectionFactory factory = new ConnectionFactory();
	    factory.setHost("localhost");
	    Connection connection = factory.newConnection();
	    Channel channel = connection.createChannel();
	
	    channel.exchangeDeclare(EXCHANGE_NAME, "topic");
	    String queueName = channel.queueDeclare().getQueue();
	
	    if (argv.length < 1) {
	      System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
	      System.exit(1);
	    }
	
	    for (String bindingKey : argv) {
	      channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
	    }
	
	    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
	
	    Consumer consumer = new DefaultConsumer(channel) {
	      @Override
	      public void handleDelivery(String consumerTag, Envelope envelope,
	                                 AMQP.BasicProperties properties, byte[] body) throws IOException {
	        String message = new String(body, "UTF-8");
	        System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
	      }
	    };
	    channel.basicConsume(queueName, true, consumer);
	  }
	}

编译并运行示例，包括教程1中的类路径，在Windows上，使用％CP％。

编译：

	javac -cp $CP ReceiveLogsTopic.java EmitLogTopic.java

接收所有日志：

	java -cp $CP ReceiveLogsTopic "#"

接收所有“kern”的日志：

	java -cp $CP ReceiveLogsTopic "kern.*"

或者你仅仅想接收'critical'日志：

	java -cp $CP ReceiveLogsTopic "*.critical"

你可以创建多个绑定：

	java -cp $CP ReceiveLogsTopic "kern.*" "*.critical"

发出一个路由关键字为"kern.critical"的日志，输入：

	java -cp $CP EmitLogTopic "kern.critical" "A critical kernel error"

注意代码没有对特定的路由和绑定关键字做臆断，你可以操作多于两个的路由关键字参数。

接下来，在教程6中，了解如何做一个往返程序作为远程过程调用消息

