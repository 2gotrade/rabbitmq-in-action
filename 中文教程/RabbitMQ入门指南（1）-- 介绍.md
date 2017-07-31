## RabbitMQ入门指南（1）--介绍

声明：本文都是参考RabbitMQ的官方指南翻译过来的，由于本人水平有限难免有翻译不当的地方，敬请谅解。

> ##### 前提条件
> 本教程假定RabbitMQ 已在标准端口（5672）上的localhost上安装并运行。如果使用不同的主机，端口或凭据，连接设置将需要调整。 

RabbitMQ是一个消息代理：它接受并转发消息。您可以将其视为邮局：当您将要发送的邮件放入邮箱中时，您可以确信邮递员最终会将邮件送到收件人的手中。RabbitMQ是就相当于一个邮箱，邮局和邮递员。

RabbitMQ和邮局之间的主要区别在于RabbitMQ不处理文件，而是接受，存储并以二进制形式将消息转发。

RabbitMQ和消息传递一般使用一些术语。

- 生产者

生产无非就是发送，发送消息的程序就是一个生产者，我们使用“P”来描述它。

![producer.png](http://www.rabbitmq.com/img/tutorials/producer.png)

- 队列

队列在RabbitMQ中的相当于邮箱，它位于RabbitMQ内部，虽然消息流通过RabbitMQ和您的应用程序，但是它们仅仅存储在队列中。一个队列仅由主机的存储器＆磁盘限制约束，它本质上是一个大的消息缓冲器。多个生产者可以通过一个队列发送消息，同样多个消费者也可以通同一个消息队列中接收消息。我们使用如下图描述一个队列：

![queue.png](http://www.rabbitmq.com/img/tutorials/queue.png)

- 消费者

消费与接收的含义相似，一个消费者通常是一个等着接受消息的程序，我们使用"C"来描述：

![consumer.png](http://www.rabbitmq.com/img/tutorials/consumer.png)

> ##### 注意
> 生产者，消费者和代理者不需要一定在同一个机器上，事实上，大多数应用程序中，他们并不在同一个机器上。

#### "Hello World"（使用Java客户端）

在本教程的这一部分，我们将使用Java编写两个程序; 一个发送单个消息的生产者，以及一个接收消息并将其打印输出的消费者。我们将会忽视掉一些Java API的细节，只专注于这个非常简单的事情，即“Hello World”的消息传递。

在下图中，“P”是我们的生产者，“C”是我们的消费者。中间的框是队列，即RabbitMQ消费者的消息缓冲区。

![python-one.png](http://www.rabbitmq.com/img/tutorials/python-one.png)

> ##### Java客户端库
> RabbitMQ 遵循[AMQP协议](http://www.amqp.org/),那是一个开放的，并且通用的消息协议，用于消息传递。本教程使用AMQP 0-9-1，它有许多不同语言的 RabbitMQ客户端。我们将使用RabbitMQ提供的Java客户端。
> 下载[客户端库](http://central.maven.org/maven2/com/rabbitmq/amqp-client/4.0.2/amqp-client-4.0.2.jar) 及其依赖项（SLF4J API和 SLF4J Simple）。根据教程将这些Java文件复制到工作目录中。
> 请注意本教程依赖SLF4J只是一个简单的例子，您还应该使用完整的日志库，如生产中的Logback。
> （RabbitMQ Java客户端也位于Maven中央存储库中，其中包含groupId com.rabbitmq和artifactId amqp-client。）

现在我们有Java客户端及其依赖关系，我们可以编写一些代码。

- 发送

![sending.png](http://www.rabbitmq.com/img/tutorials/sending.png)

我们将会让消息发布者（发件人）发送消息，我们的消息消费者（接收者）接收消息。发布者连接到RabbitMQ上，发送一个简单的消息，然后退出。

在 [Send.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)中，我们需要一些导入的类：

	import com.rabbitmq.client.ConnectionFactory;
	import com.rabbitmq.client.Connection;
	import com.rabbitmq.client.Channel;

建立这个类，为队列命名：

	public class Send {
	  private final static String QUEUE_NAME = "hello";
	  public static void main(String[] argv)
	      throws java.io.IOException {
	      ...
	  }
	}

接着，我们创建一个到服务器的连接：

	ConnectionFactory factory = new ConnectionFactory();
	factory.setHost("localhost");
	Connection connection = factory.newConnection();
	Channel channel = connection.createChannel();

这个是抽象的socket连接，对我们进行协议版本协商和认证等。 这里我们连接到本地机器上的代理，因此它是localhost。如果我们想连接到不同机器上的代理，只需要说明它的主机名和IP地址。

接下来，我们创建一个通道，这是大部分用于完成任务的API所在的地方。

对于发送，我们必须声明一个发送队列，然后我们把消息发送到这个队列上：

	channel.queueDeclare（QUEUE_NAME，false，false，false，null）; 
	String message = “Hello World！” ; 
	channel.basicPublish（“”，QUEUE_NAME，null，message.getBytes（））; 
	System.out.println（“[x] Sent'” + message + “'”）;

声明一个队列是幂等的，仅仅在要声明的队列不存在时才创建。消息内容是二进制数组，所以你可以随你喜好编码。

最后，我们关闭通道和连接:

	channel.close();
	connection.close();

这是整个[Send.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)类。

> ##### 发送没有起作用!
> 如果你是第一次使用RabbitMQ并且你没有看到"Sent"消息，你可能抓耳挠腮的想到底是哪里出的问题。可能是代理启动时没有足够空间(默认它需要至少1Gb 空间)，因此拒绝接受消息。通过检查代理的日志文件来确定这个问题，必要情况下可以降低限制大小。配置文件的文档将会告诉你怎样设置disk_free_limit。

- 接收

上面代码是构建我们的发送者。我们的接收者是从RabbitMQ中提取消息，所以不像发送者那样发送一个简单的消息，我们需要一直运行监听消息并且输出消息。
![receiving.png](http://www.rabbitmq.com/img/tutorials/receiving.png)
在[Recv.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)中的代码有与Send中几乎相同的引用:

	import com.rabbitmq.client.ConnectionFactory;
	import com.rabbitmq.client.Connection;
	import com.rabbitmq.client.Channel;
	import com.rabbitmq.client.Consumer;
	import com.rabbitmq.client.DefaultConsumer;


这额外的DefaultConsumer是一个实现Consumer 接口的类，用于缓冲由服务器推送给我们的消息。

跟创建发送者相同，我们打开一个连接和一个通道，声明一个我们要消费的队列。注意要与发送的队列相匹配。

	public class Recv {
	  private final static String QUEUE_NAME = "hello";
	  public static void main(String[] argv) throws java.io.IOException, java.lang.InterruptedException {
		ConnectionFactory factory = new ConnectionFactory();
	    factory.setHost("localhost");
	    Connection connection = factory.newConnection();
	    Channel channel = connection.createChannel();
	    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
	    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
	      ...
	  }
	}

注意我们在这里同样声明了一个队列。因为我们可能在发送者之前启动接收者，所以我们要确保队列存在，然后再尝试从中消费消息。 

我们即将告诉服务器将队列中的消息传递给我们。由于它会异步地推送我们的邮件，所以我们提供一个对象形式的回调，缓冲消息，直到我们准备好使用它们。这是一个DefaultConsumer子类。

	Consumer consumer = new DefaultConsumer(channel) {
	  @Override
	  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
	      throws IOException {
	    String message = new String(body, "UTF-8");
	    System.out.println(" [x] Received '" + message + "'");
	  }
	};
	channel.basicConsume(QUEUE_NAME, true, consumer);

这是整个[Recv.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)类。

- 把它们放在一起

您可以在类路径上只使用RabbitMQ java客户端来编译这两个：

	javac -cp amqp-client-4.0.2.jar Send.java Recv.java


为了运行它们，您需要使用rabbitmq-client.jar及其对类路径的依赖。在终端中，运行消费者（接收者）：

	java -cp .:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar Recv

然后，运行发布者（发送者）：

	java -cp .:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar Send

在windows环境中，使用分号而不是冒号分隔类路径中的项目。

消费者将通过RabbitMQ打印从发布商获得的消息。消费者将继续运行，等待消息（使用Ctrl-C停止它），所以尝试从另一个终端运行发布者。

> ##### 列出队列

> 您可能希望看到RabbitMQ有什么队列和其中有多少个消息。您可以使用rabbitmqctl工具（sudo root）

	sudo rabbitmqctl list_queues

> 在Windows上，省略sudo：

	rabbitmqctl.bat list_queues


接下来将进入第二部分,构建一个简单的工作队列。

> ##### 提示
> 为了保存输入，你可以将类路径设置到环境变量中

	export CP=.:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar
	java -cp $CP Send
	
> 或者在 Windows环境中:

	set CP=.;amqp-client-4.0.2.jar;slf4j-api-1.7.21.jar;slf4j-simple-1.7.22.jar
	java -cp %CP% Send