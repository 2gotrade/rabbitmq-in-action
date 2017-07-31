# RabbitMQ入门指南（2）--工作队列

**声明**：本文都是参考RabbitMQ的官方指南翻译过来的，由于本人水平有限难免有翻译不当的地方，敬请谅解。

> **前提条件**

> 本教程假定RabbitMQ 已在标准端口（5672）上的localhost上安装并运行。如果使用不同的主机，端口或凭据，连接设置将需要调整。 
 
## 工作队列（使用Java客户端）

![python-two](http://www.rabbitmq.com/img/tutorials/python-two.png)

在[第一个教程](http://www.rabbitmq.com/tutorials/tutorial-one-java.html)中，我们编写了程序来发送和接收来自命名队列的消息。在这一个中，我们将创建一个工作队列，用于在多个工作人员之间分配耗时的任务。

工作队列（又称：任务队列）背后的主要思想是避免立即执行资源密集型任务，并且必须等待完成。相反，我们安排后续完成的任务。我们将任务封装 成消息，并将其发送到队列。在后台运行的工作进程将弹出任务并最终执行作业。当你运行很多工作人员时，这些任务将在它们之间共享。

这个概念在Web应用程序中特别有用，在短时间HTTP请求窗口中无法处理复杂的任务。

### 准备

在本教程的前面部分，我们发送了一个包含“Hello World！”的消息。现在我们将要发送一些字符串，用来代表复杂的任务。我们没有一个真实的任务，比如图片的调整大小或者pdf文件渲染，所以我们通过Thread.sleep()函数，伪装一个我们是很忙景象。我们将会把字符串中点的数量来代表它的复杂度；每一个点将要花费一秒的工作。例如，一个使用Hello...描述的假任务会发送三秒。

我们将稍微修改我们前面的例子中的Send.java代码，其允许任意的消息可以通过命令行发出。这个程序会将任务安排到我们的工作队列中，所以让我们命名为  NewTask.java：

	String message = getMessage(argv);
	
	channel.basicPublish("", "hello", null, message.getBytes());
	System.out.println(" [x] Sent '" + message + "'");
	
一些帮助从命令行中获取消息参数：

	private static String getMessage(String[] strings){
	    if (strings.length < 1)
	        return "Hello World!";
	    return joinStrings(strings, " ");
	}
	
	private static String joinStrings(String[] strings, String delimiter) {
	    int length = strings.length;
	    if (length == 0) return "";
	    StringBuilder words = new StringBuilder(strings[0]);
	    for (int i = 1; i < length; i++) {
	        words.append(delimiter).append(strings[i]);
	    }
	    return words.toString();
	}

我们的老Recv.java程序也需要进行一些更改：它需要将消息体中每个点伪装成一秒。从队列中获取消息并执行任务，所以让我们称之为Worker.java：

	final Consumer consumer = new DefaultConsumer(channel) {
	  @Override
	  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
	    String message = new String(body, "UTF-8");
	
	    System.out.println(" [x] Received '" + message + "'");
	    try {
	      doWork(message);
	    } finally {
	      System.out.println(" [x] Done");
	    }
	  }
	};
	boolean autoAck = true; // acknowledgment is covered below
	channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
	
我们伪装的任务中冒充执行时间：

	private static void doWork(String task) throws InterruptedException {
	    for (char ch: task.toCharArray()) {
	        if (ch == '.') Thread.sleep(1000);
	    }
	}  

按照第一部分指南中那样编译它们（jar 文件需要再工作路径上）：

	javac -cp $CP NewTask.java Worker.java

### 循环分派

使用任务队列的优点之一是能够轻松地并行工作。如果我们正在处理一些堆积的文件的话，我们仅仅需要增加更多的工作者，通过这种方式我们是容易扩展的。

首先，让我们试着在同一时间运行两个Worker实例。他们都会从队列中获取消息，但是具体怎样做呢？让我们一起来看一看。

你需要三个打开的控制平台，其中两个用来运行工作者程序。他们将会是我们的两个消费者-C1和C2。

	# shell 1
	java -cp $CP Worker
	# => [*] Waiting for messages. To exit press CTRL+C

	# shell 2
	java -cp $CP Worker
	# => [*] Waiting for messages. To exit press CTRL+C

在这第三个控制平台我们用来发布新的任务。一旦你启动消费者，你就可以发布消息了：

	# shell 3
	java -cp $CP NewTask
	# => First message.
	java -cp $CP NewTask
	# => Second message..
	java -cp $CP NewTask
	# => Third message...
	java -cp $CP NewTask
	# => Fourth message....
	java -cp $CP NewTask
	# => Fifth message.....	

让我们看看什么被投递到我们工作者那里：

	java -cp $CP Worker
	# => [*] Waiting for messages. To exit press CTRL+C
	# => [x] Received 'First message.'
	# => [x] Received 'Third message...'
	# => [x] Received 'Fifth message.....'

	java -cp $CP Worker
	# => [*] Waiting for messages. To exit press CTRL+C
	# => [x] Received 'Second message..'
	# => [x] Received 'Fourth message....'

默认情况下，RabbitMQ将按顺序将每条消息发送给下一个消费者。平均每个消费者将获得相同数量的消息。这种分发消息的方式被称为轮询（round-robin）。可以尝试三名或更多的工作者。

### 消息确认

执行一个任务可能需要几秒钟。你可能会想，如果一个消费者开始一个需要很长时间的任务，并且在处理完成部分的情况下就死掉了会发生什么情况。就我们当前的代码来说，一旦RabbitMQ将消息传递给消费者，它就会立即将消息从内存中删除。在这种情况下，如果你杀掉一个正在处理的工作者你会丢失它正在处理的消息。我们也同时失去了已经分配给这个工作者并且尚未处理的消息。

但是我们不想丢失任何任务，如果一个工作者死掉，我们期望将任务传递给另一个工作者。

一个消息确认是由消费者发出，告诉RabbitMQ这个消息已经被接受，处理完成，RabbitMQ 可以删除它了。

如果消费者死机（其通道关闭，连接关闭或TCP连接丢失），而没有发送确认信息，RabbitMQ将会认定这个消息没有完全处理成功，如果同时有其他消费者在线，则会迅速将其重新提供给另一个消费者。通过这种方式，即使工作者有时会死掉，你依旧可以保证没有消息会被丢失。

 这里不存在消息超时;RabbitMQ只会在工作者连接死掉才重新传递这个消息。即使一个消息要被处理很长很长时间，也不是问题。

消息确认机制默认情况下是开着的。在前面的例子中，我们通过autoAck = true 标志明确地将它们关闭。现在将此标志设置为false，一旦我们完成一个任务，工作者需要发送一个确认信号。

	channel.basicQos(1); // accept only one unack-ed message at a time (see below)
	
	final Consumer consumer = new DefaultConsumer(channel) {
	  @Override
	  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
	    String message = new String(body, "UTF-8");
	
	    System.out.println(" [x] Received '" + message + "'");
	    try {
	      doWork(message);
	    } finally {
	      System.out.println(" [x] Done");
	      channel.basicAck(envelope.getDeliveryTag(), false);
	    }
	  }
	};
	boolean autoAck = false;
	channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);

使用这段代码，我们可以保证即使你将一个正在处理消息的工作者通过CTRL+C来终止它运行，依旧没有消息会丢失。稍后，工作者死亡后没有发送确认的消息会被重新传递。

> **忘记确认**

> 忘记basicAck是一个常见的错误，但后果是严重的。当您的客户端退出时，消息将被重新传递（可能看起来像随机重新传递），RabbitMQ将会消耗越来越多的内存，因为它将无法释放那些没有发送确认的消息。

> 为了调试这种错误，您可以使用rabbitmqctl 打印messages_unacknowledged字段：

	sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged

> 在Windows上:

	rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged

### 消息持久化

我们已经学习了如何在确定消费者是否已经死掉，并且保证任务不会丢失。但是如果RabbitMQ服务器停止，我们的任务仍然会丢失。

当RabbitMQ退出或崩溃时，它会忘记队列和消息，除非你告诉它不要这样做。两个事情需要做来保证消息不会丢失：我们标记队列和消息持久化。

首先，我们需要确保RabbitMQ不会失去我们的队列。为了这样，我们需要将其声明为持久的：

	boolean durable = true;
	channel.queueDeclare("hello", durable, false, false, null);

虽然这个命令本身是正确的，但它不会立即在我们的程序里运行。这是因为我们已经定义了一个 不持久化且名为hello的队列。RabbitMQ不允许您使用不同的参数重新定义一个存在的队列，并会向尝试执行此操作的任何程序返回错误。但是有一个快速的解决方，就是 声明一个不同名字的队列，比如task_queue:：

	boolean durable = true;
	channel.queueDeclare("task_queue", durable, false, false, null);

这个queueDeclare的更改需要应用于生产者和消费者的代码中。

在这点上，我们可以保证即使RabbitMQ重启，task_queue队列也不会丢失。现在我们需要通过设置MessageProperties(实现了BasicProperties)的值为PERSISTENT_TEXT_PLAIN来标记消息持久化。

	import com.rabbitmq.client.MessageProperties; 
	
	channel.basicPublish（“”，“task_queue”，
	            MessageProperties.PERSISTENT_TEXT_PLAIN，
	            message.getBytes（））;
	

> **注意消息持久化**

> 将消息标记为持久化不能完全保证消息不会丢失，虽然这样会告诉RabbitMQ保存消息到硬盘上。但是当RabbitMQ接受消息并且还没有保存时​​，仍然有一个很短的时间窗口。此外，RabbitMQ不能让每个消息同步,它可能仅仅保存在缓存中，还没有真正的写入到硬盘中。这持久化的保证不是健壮的，但是对我们的简单的任务队列来说已经足够了。如果你需要更健壮的持久化保证，你可以与[发布商确认](https://www.rabbitmq.com/confirms.html)。

### 公平分发

您可能已经注意到，分发过程并没有如我们想的那样运作。例如，在某一种情况下有两个工作者，当所有奇数消息是很多的并且所有偶数的是少量的，一个工作者会一直忙碌下去，而另一个则会几乎不做什么事情。那么，RabbitMQ不会在意那个事情，它会一直均匀的分发消息。

这种情况发生因为RabbitMQ仅仅分发消息到队列中。它不关心消费者的未确认消息的数量。它仅仅盲目地将N个消息发送到N个消费者。

![prefetch-count](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

为了解决这个问题，我们可以使用basicQos方法，设置prefetchCount=1。这样将会告知RabbitMQ不要同时给一个工作者超过一个任务，或者换句话说在一个工作者处理完成，发送确认之前不要给它分发一个新的消息。相反，它将把消息分发到下一个不繁忙的工作者。

	int prefetchCount = 1 ; 
	channel.basicQos（prefetchCount）;

> **注意队列大小**

> 如果所有的工作者都是在忙碌的，你的队列就会被填满。你将会想关注这件事，可能要添加更多的工作者，或者有些其他策略。

### 把它们放在一起

我们的[NewTask.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/NewTask.java)最终代码：

	import java.io.IOException;
	import com.rabbitmq.client.ConnectionFactory;
	import com.rabbitmq.client.Connection;
	import com.rabbitmq.client.Channel;
	import com.rabbitmq.client.MessageProperties;
	
	public class NewTask {
	
	  private static final String TASK_QUEUE_NAME = "task_queue";
	
	  public static void main(String[] argv)
	                      throws java.io.IOException {
	
	    ConnectionFactory factory = new ConnectionFactory();
	    factory.setHost("localhost");
	    Connection connection = factory.newConnection();
	    Channel channel = connection.createChannel();
	
	    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
	
	    String message = getMessage(argv);
	
	    channel.basicPublish( "", TASK_QUEUE_NAME,
	            MessageProperties.PERSISTENT_TEXT_PLAIN,
	            message.getBytes());
	    System.out.println(" [x] Sent '" + message + "'");
	
	    channel.close();
	    connection.close();
	  }      
	  //...
	}

我们的[Worker.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Worker.java)代码:

	import com.rabbitmq.client.*;
	
	import java.io.IOException;
	
	public class Worker {
	  private static final String TASK_QUEUE_NAME = "task_queue";
	
	  public static void main(String[] argv) throws Exception {
	    ConnectionFactory factory = new ConnectionFactory();
	    factory.setHost("localhost");
	    final Connection connection = factory.newConnection();
	    final Channel channel = connection.createChannel();
	
	    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
	    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
	
	    channel.basicQos(1);
	
	    final Consumer consumer = new DefaultConsumer(channel) {
	      @Override
	      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
	        String message = new String(body, "UTF-8");
	
	        System.out.println(" [x] Received '" + message + "'");
	        try {
	          doWork(message);
	        } finally {
	          System.out.println(" [x] Done");
	          channel.basicAck(envelope.getDeliveryTag(), false);
	        }
	      }
	    };
	    boolean autoAck = false;
	    channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
	  }
	
	  private static void doWork(String task) {
	    for (char ch : task.toCharArray()) {
	      if (ch == '.') {
	        try {
	          Thread.sleep(1000);
	        } catch (InterruptedException _ignored) {
	          Thread.currentThread().interrupt();
	        }
	      }
	    }
	  }
	}

使用消息确认和prefetchCount可以设置工作队列。即使RabbitMQ重新启动，耐久性选项也让任务生存下去。

有关Channel方法和MessageProperties的更多信息，可以在线浏览 [JavaDocs](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/)。

现在我们可以继续阅读教程3，并学习如何向许多消费者提供相同的消息。

