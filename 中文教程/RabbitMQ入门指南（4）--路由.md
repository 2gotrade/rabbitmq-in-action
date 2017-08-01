# RabbitMQ入门指南（4）--路由

**声明**：本文都是参考RabbitMQ的官方指南翻译过来的，由于本人水平有限难免有翻译不当的地方，敬请谅解。

> **前提条件**

> 本教程假定RabbitMQ 已在标准端口（5672）上的localhost上安装并运行。如果使用不同的主机，端口或凭据，连接设置将需要调整。 

*摘要: Routing，有选择性地接收消息*

## 路由（使用Java客户端）

在上一个教程中，我们构建了一个简单的日志系统 ，我们可以将我们的日志信息广播到多个接收者。

在本教程中，我们将为其添加一个功能 ，让它可以仅订阅一部分消息。例如，我们将能够仅将关键的错误消息引导到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。

- 绑定

在之前的例子里我们已经创建绑定。你可以回顾下代码：

	channel.queueBind(queueName, EXCHANGE_NAME, "");

绑定是交换和队列之间的关系。这可以简单地理解为：队列对来自此交换器的消息感兴趣。

绑定可以带上额外的路由关键字参数。为了消除对basic_publish参数混淆，我们将其称之为绑定关键字。以下是我们如何通过一个关键字创建一个绑定：

	channel.queueBind(queueName, EXCHANGE_NAME, "black");

这绑定关键字的意义取决于 交换机类型。我们之前使用的那个fanout 交换器忽略它的值。

### 直接交换

我们上一个教程的日志记录系统将所有消息广播到所有消费者。我们想扩展它，让其允许依据其严格的规则过滤消息。例如我们可能想让一个往硬盘中写日志消息的程序仅仅接收关键的错误，而不是将硬盘空间浪费在警告和信息的日志消息上。 

我们使用fanout类型的交换器，那个不会给我们太多的灵活性-它仅仅能胜任没头脑的广播。

我们可以使用direct类型的交换器来替代。一个direct交换器背后的路由算法是简单的-消息传递到绑定关键字与消息中路由关键字完全匹配的队列中。

为了说明，考虑接下来结构：

![direct-exchange.png](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)

在这个结构里，我们看见了这direct类型的交换器绑定了两个队列。第一个队列装有orange绑定关键字，第二个有两个绑定，一个是black绑定关键字并且另一个是green关键字。 

在这个结构里，发送到交换器里的消息，其中消息中带路由关键字为orange将要路由到队列Q1上，消息中带路由关键字为black或green将路由到队列Q2上。所有其他类型的消息会被丢弃。

### 多重绑定

![direct-exchange-multiple.png](http://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

将一个绑定关键字绑定到多个队列是完全合法的。在我们例子中使用绑定关键字black将X和Q1绑定在一起。在这种情况下，direct类型的交换器与fanout类型相似，同样会广播这消息到所有符合的队列中。一个路由关键位balck的关键字将会被传递到Q1和Q2。

### 发出日志

我们将会为我们的日志系统使用这个模型。使用direct类型的交换器来代替fanout类型，发送消息。由于这路由关键字我们可以严格的记录。接收程序通过这种方式可以严格接收它想接收的。让我们首先关注发布日志。

 总之，我们首先需要创建个交换机。

	channel.exchangeDeclare(EXCHANGE_NAME, "direct");

我们准备发送一个消息：

	channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());

为了简化这个事情，我们保证这severity是inof,warning,error中的一个。

### 订阅

接收消息将像上一个教程一样工作，有一个例外，我们会把每一个我们感兴趣的severity创建一个新的绑定。

	String queueName = channel.queueDeclare().getQueue();
	
	for(String severity : argv){
	  channel.queueBind(queueName, EXCHANGE_NAME, severity);
	}

### 把它们放在一起

![python-four.png](http://www.rabbitmq.com/img/tutorials/python-four.png)

[EmitLogDirect.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLogDirect.java)类的代码：

	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	
	public class EmitLogDirect {
	
	    private static final String EXCHANGE_NAME = "direct_logs";
	
	    public static void main(String[] argv)
	                  throws java.io.IOException {
	
	        ConnectionFactory factory = new ConnectionFactory();
	        factory.setHost("localhost");
	        Connection connection = factory.newConnection();
	        Channel channel = connection.createChannel();
	
	        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
	
	        String severity = getSeverity(argv);
	        String message = getMessage(argv);
	
	        channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
	        System.out.println(" [x] Sent '" + severity + "':'" + message + "'");
	
	        channel.close();
	        connection.close();
	    }
	    //..
	}

[ReceiveLogsDirect.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogsDirect.java)类的代码：

	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	
	public class ReceiveLogsDirect {
	
	  private static final String EXCHANGE_NAME = "direct_logs";
	
	  public static void main(String[] argv) throws Exception {
	    ConnectionFactory factory = new ConnectionFactory();
	    factory.setHost("localhost");
	    Connection connection = factory.newConnection();
	    Channel channel = connection.createChannel();
	
	    channel.exchangeDeclare(EXCHANGE_NAME, "direct");
	    String queueName = channel.queueDeclare().getQueue();
	
	    if (argv.length < 1){
	      System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
	      System.exit(1);
	    }
	
	    for(String severity : argv){
	      channel.queueBind(queueName, EXCHANGE_NAME, severity);
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

像往常一样编译（参见教程一编译和类路径建议）。为方便起见，我们将在运行示例时使用环境变量$ CP（Windows上的％CP％）作为类路径。

	javac -cp $CP ReceiveLogsDirect.java EmitLogDirect.java

如果你想仅保存warning和error记录不包含info记录信息到文件里，打开一个控制台并输入：

	java -cp $CP ReceiveLogsDirect warning error > logs_from_rabbit.log

如果你想在你的屏幕上看所有的日志信息，打开一个新的终端并键入：

	java -cp $CP ReceiveLogsDirect info warning error
	# => [*] Waiting for logs. To exit press CTRL+C

例如，为了发布一个错误日志信息，仅需要键入：

	java -cp $CP EmitLogDirect error "Run. Run. Or it will explode."
	# => [x] Sent 'error':'Run. Run. Or it will explode.'

接下来到指南的第五部分，了解如何根据模式来监听消息。








