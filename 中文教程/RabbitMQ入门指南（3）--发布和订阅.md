# RabbitMQ入门指南（3）--发布和订阅

**声明**：本文都是参考RabbitMQ的官方指南翻译过来的，由于本人水平有限难免有翻译不当的地方，敬请谅解。

> **前提条件**

> 本教程假定RabbitMQ 已在标准端口（5672）上的localhost上安装并运行。如果使用不同的主机，端口或凭据，连接设置将需要调整。 

*摘要: 一次将消息发送到多个消费者*

## 发布和订阅（使用Java客户端）

在[上一个教程](http://www.rabbitmq.com/tutorials/tutorial-two-java.html)中，我们创建了一个工作队列。工作队列背后的假设是每个任务都被准确的传递给工作者。在这部分中，我们将会做一些完全不同的事--我们将一个消息传递给多个消费者。这种模式被称为“发布/订阅”。

为了说明这个模式，我们会建立一个简单的日志记录系统。它将包括两个程序：第一个将发出日志消息，第二个将接收并打印它们。

在我们的日志系统中，每一个运行的接收者拷贝程序将会获得信息。通过这个方式我们可以运行一个接收者，直接的把日志记录到硬盘中；在同一时间我们可以运行另一个接收者，在屏幕上看这些日志。 

本质上，发布日志消息等同于广播到所有接收者。

### 交换

在本教程的前面部分，我们将消息发送到队列里，并从队列中接收消息。现在是时候介绍RabbitMQ中全消息模型。

让我们快速温习一下在先前的教程中介绍的内容：

一个发送消息的生产者是一个用户程序。 一个存储消息的队列是一个缓冲。 一个接收消息的消费者是一个用户程序。

 在RabbitMQ消息模型中核心的思想是生产者从不将任何消息直接发送到队列。实际上，生产者通常甚至不知道是否将消息传递到任何队列。
 
相反，生产者仅能将消息发送到一个交换机。一个交换机是一个非常简单的事物。一方面，它从生产者那里接收消息，另一方将消息推送到队列中。这个交换机必须清楚地知道它所接收到的消息要如何处理。是否将它附加到一个特别的队列中？是否将它附加到多个队列中？或者是否它应该被丢弃。规则的定义是由交换类型决定的。

![exchanges.png](http://www.rabbitmq.com/img/tutorials/exchanges.png)

**有几个交换类型：direct，topic，deaders，fanout**

我们来关注最后一个fanout。让我们创建一个这种类型的交换机并且称呼它为logs:

	channel.exchangeDeclare("logs", "fanout");

这fanout交换机是非常简单的。正如您可以从名称猜测的，它只是将所有收到的消息广播到所有知道的队列。这个正是我们的记录器所需要的。

> **交换机列表**

> 要列出服务器上的交换机，您可以运行有用的rabbitmqctl：

	sudo rabbitmqctl list_exchanges

> 在这个列表里有一些以amq.打头的交换机和默认（未命名）的交换机。这些是默认创建的，但是不太可能你会在某个时刻使用它们。 

> **匿名交换机**

> 在本教程的前面部分，我们对交换机毫无了解，但仍然能够将消息发送到队列。这是可能的，因为我们使用默认交换，我们通过空字符串（""）标识。

>  回想一下我们以前是如何发送消息的：

	channel.basicPublish("", "hello", null, message.getBytes());

这第一个参数是交换机的名字。空字符串说明它是默认的或者匿名的交换机：路由关键字存在的话，消息通过路由关键字的名字路由到特定的队列上。

现在，我们可以发布我们自己命名的交换机：

	channel.basicPublish( "logs", "", null, message.getBytes());

### 临时队列

你可能会想起先前我们使用的队列是有特定的名字的（是否记得hello和task_queue）。命名一个队列对我们来说是至关重要的--我们需要指定工作者到这相同的队列上。当您想要在生产者和消费者之间共享队列时，给队列名是重要的。

 但是我们的记录器不是这样。我们希望监听所有的日志消息，而不仅仅是它们的一部分。我们也只对当前流行的消息不感兴趣。为了解决这样的问题，我们需要做两件事。 
 
 首先，无论我们什么时候连接RabbitMQ，我们都需要一个新的空队列。为此，我们可以创建一个具有随机名称的队列或者让服务器为我们选择一个随机队列名称。
 
 其次，一旦我们断开消费者，队列应该被自动删除。
 
  在Java客户端里，当我们使用无参数调用queueDeclare()方法，我们创建一个自动产生的名字，非持久化，独占的，自动删除的队列。

	String queueName = channel.queueDeclare().getQueue();
 	
此时，queueName包含一个随机队列名称。例如，它可能看起来像amq.gen-JzTY20BRgKO-HjmUJj0wLg。

### 绑定

![bindings.png](http://www.rabbitmq.com/img/tutorials/bindings.png)

我们已经创建了一个fanout交换机和队列。现在我们需要告诉交换机发送消息到我们的队列上。这种交换机和队列之间的关系称之为一个绑定。

	channel.queueBind(queueName, "logs", "");

从现在开始，日志交换机将附加消息到我们的队列中。

> **绑定列表**

> 你可以使用rabbitmqctl list_bindings列出存在的绑定。

### 把它们放在一起

![python-three-overall.png](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

这发送日志消息的生产者程序上一个教程并没有太大的区别。最重要的变化是我们现在想将消息发布到我们的日志交换机，而不是无名的。发送时需要提供一个routingKey，但是对于fanout交换机来说，它的值会被忽略。以下是[EmitLog.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLog.java)程序的代码：

	import java.io.IOException;
	import com.rabbitmq.client.ConnectionFactory;
	import com.rabbitmq.client.Connection;
	import com.rabbitmq.client.Channel;
	
	public class EmitLog {
	
	    private static final String EXCHANGE_NAME = "logs";
	
	    public static void main(String[] argv)
	                  throws java.io.IOException {
	
	        ConnectionFactory factory = new ConnectionFactory();
	        factory.setHost("localhost");
	        Connection connection = factory.newConnection();
	        Channel channel = connection.createChannel();
	
	        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
	
	        String message = getMessage(argv);
	
	        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
	        System.out.println(" [x] Sent '" + message + "'");
	
	        channel.close();
	        connection.close();
	    }
	    //...
	}

如你所见，建立连接后，我们声明一个交换机。这个步骤是必须的，因为发布到一个不存在的交换机是禁止的。

如果没有任何队列绑定到交换机，消息将丢失，但是对我们来说没关系; 如果没有消费者正在监听，我们可以安全的丢弃消息。 

[ReceiveLogs.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogs.java)代码：

	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	
	public class ReceiveLogs {
	  private static final String EXCHANGE_NAME = "logs";
	
	  public static void main(String[] argv) throws Exception {
	    ConnectionFactory factory = new ConnectionFactory();
	    factory.setHost("localhost");
	    Connection connection = factory.newConnection();
	    Channel channel = connection.createChannel();
	
	    channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
	    String queueName = channel.queueDeclare().getQueue();
	    channel.queueBind(queueName, EXCHANGE_NAME, "");
	
	    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
	
	    Consumer consumer = new DefaultConsumer(channel) {
	      @Override
	      public void handleDelivery(String consumerTag, Envelope envelope,
	                                 AMQP.BasicProperties properties, byte[] body) throws IOException {
	        String message = new String(body, "UTF-8");
	        System.out.println(" [x] Received '" + message + "'");
	      }
	    };
	    channel.basicConsume(queueName, true, consumer);
	  }
	}

像以前一样编译，我们就 完成了。

	javac -cp $CP EmitLog.java ReceiveLogs.java

如果要将日志保存到文件，只需打开控制台并键入：

	java -cp $CP ReceiveLogs > logs_from_rabbit.log

如果您希望在屏幕上看到日志，则新建一个新的终端并运行：

	java -cp $CP ReceiveLogs

当然，为了发出日志键入：

	java -cp $CP EmitLog

使用rabbitmactl list_bindings你可以验证这代码确实创建绑定和我们想要的队列。随着两个ReceiveLogs.java程序的运行你可以看到一些如：

	sudo rabbitmqctl list_bindings
	 ＃=>列出绑定... 
	＃=>日志交换amq.gen-JzTY20BRgKO-HjmUJj0wLg队列[] 
	＃=>日志交换amq.gen-vso0PVvyiRIL2WoV3i48Yg队列[] 
	＃=> ...完成。

这个结果的解释很简单：来自交换机的日志的数据转到具有服务器分配名称的两个队列。这正是我们的意图。 

要了解如何监听一个消息的子集，让我们继续指南的第四部分。







