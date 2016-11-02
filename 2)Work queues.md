
##工作队列

**（使用pika 0.9.5 Python客户端）**

![](http://www.rabbitmq.com/img/tutorials/python-two.png)  

在第一篇教程中，我们已经写了一个从已知队列中发送和获取消息的程序。在这篇教程中，我们将创建一个工作队列（Work Queue），它会发送一些耗时的任务给多个工作者（Worker）。

工作队列（又称：任务队列——Task Queues）是为了避免等待一些占用大量资源、时间的操作。当我们把任务（Task）当作消息发送到队列中，一个运行在后台的工作者（worker）进程就会取出任务然后处理。当你运行多个工作者（workers），任务就会在它们之间共享。

这个概念在网络应用中是非常有用的，它可以在短暂的HTTP请求中处理一些复杂的任务。

##准备

之前的教程中，我们发送了一个包含“Hello World!”的字符串消息。现在，我们将发送一些字符串，把这些字符串当作复杂的任务。我们没有真实的例子，例如图片缩放、pdf文件转换。所以使用time.sleep()函数来模拟这种情况。我们在字符串中加上点号（.）来表示任务的复杂程度，一个点（.）将会耗时1秒钟。比如"Hello..."就会耗时3秒钟。

我们对之前教程的Send.java做些简单的调整，以便可以发送随意的消息。这个程序会按照计划发送任务到我们的工作队列中。我们把它命名为NewTask.java：

```java
String message = getMessage(argv);

channel.basicPublish("", "hello", null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```

我们从终端(command line argument)获取要传递的消息:

```java
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

```

我们的旧类（Recv.java）同样需要做一些改动：它需要为消息体中每一个点号（.）模拟1秒钟的操作。它会从队列中获取消息并执行，我们把它命名为Worker.java：

```java
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
```

模拟任务执行:

```java
private static void doWork(String task) throws InterruptedException {
    for (char ch: task.toCharArray()) {
        if (ch == '.') Thread.sleep(1000);
    }
}
```

编译它们:

```
$ javac -cp rabbitmq-client.jar NewTask.java Worker.java
```

##循环调度:

使用工作队列的一个好处就是它能够并行的处理队列。如果堆积了很多任务，我们只需要添加更多的工作者（workers）就可以了，扩展很简单。

首先，我们先同时运行两个worker.py脚本，它们都会从队列中获取消息，到底是不是这样呢？我们看看。

你需要打开三个终端，两个用来运行worker.py脚本，这两个终端就是我们的两个消费者（consumers）—— C1 和 C2。

```shell
shell1$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
Worker
 [*] Waiting for messages. To exit press CTRL+C
```

```shell
shell2$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
Worker
 [*] Waiting for messages. To exit press CTRL+C

```
第三个终端，我们用来发布新任务。你可以发送一些消息给消费者（consumers）：

```shell
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask First message.
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Second message..
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Third message...
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Fourth message....
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Fifth message.....
```
看看到底发送了什么给我们的工作者（workers）：

```shell
shell1$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
Worker
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'First message.'
 [x] Received 'Third message...'
 [x] Received 'Fifth message.....'
```

```shell
shell2$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
Worker
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Second message..'
 [x] Received 'Fourth message....'
```
默认来说，RabbitMQ会按顺序得把消息发送给每个消费者（consumer）。平均每个消费者都会收到同等数量得消息。这种发送消息得方式叫做——轮询（round-robin）。试着添加三个或更多得工作者（workers）。

##消息确认

当处理一个比较耗时得任务的时候，你也许想知道消费者（consumers）是否运行到一半就挂掉。当前的代码中，当消息被RabbitMQ发送给消费者（consumers）之后，马上就会在内存中移除。这种情况，你只要把一个工作者（worker）停止，正在处理的消息就会丢失。同时，所有发送到这个工作者的还没有处理的消息都会丢失。

我们不想丢失任何任务消息。如果一个工作者（worker）挂掉了，我们希望任务会重新发送给其他的工作者（worker）。

为了防止消息丢失，RabbitMQ提供了消息响应（acknowledgments）。消费者会通过一个ack（响应），告诉RabbitMQ已经收到并处理了某条消息，然后RabbitMQ就会释放并删除这条消息。

如果消费者（consumer）挂掉了，没有发送响应，RabbitMQ就会认为消息没有被完全处理，然后重新发送给其他消费者（consumer）。这样，及时工作者（workers）偶尔的挂掉，也不会丢失消息。

消息是没有超时这个概念的；当工作者与它断开连的时候，RabbitMQ会重新发送消息。这样在处理一个耗时非常长的消息任务的时候就不会出问题了。

消息响应默认是开启的。之前的例子中我们可以使用no_ack=True标识把它关闭。是时候移除这个标识了，当工作者（worker）完成了任务，就发送一个响应。

```java
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
```
运行上面的代码，我们发现即使使用CTRL+C杀掉了一个工作者（worker）进程，消息也不会丢失。当工作者（worker）挂掉这后，所有没有响应的消息都会重新发送。

> #### 忘记确认

> 一个很容易犯的错误就是忘了basic_ack，后果很严重。消息在你的程序退出之后就会重新发送，如果它不能够释放没响应的消息，RabbitMQ就会占用越来越多的内存。

> 为了排除这种错误，你可以使用rabbitmqctl命令，输出messages_unacknowledged字段：

> ```
> $ sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> Listing queues ...
> hello    0       0
> ...done.
> ```

##消息持久化

如果你没有特意告诉RabbitMQ，那么在它退出或者崩溃的时候，将会丢失所有队列和消息。为了确保信息不会丢失，有两个事情是需要注意的：我们必须把“队列”和“消息”设为持久化。

首先，为了不让队列消失，需要把队列声明为持久化（durable）：

    boolean durable = true;
	channel.queueDeclare("hello", durable, false, false, null);

尽管这行代码本身是正确的，但是仍然不会正确运行。因为我们已经定义过一个叫hello的非持久化队列。RabbitMq不允许你使用不同的参数重新定义一个队列，它会返回一个错误。但我们现在使用一个快捷的解决方法——用不同的名字，例如task_queue。

	boolean durable = true;
	channel.queueDeclare("task_queue", durable, false, false, null);
	
这个queue_declare必须在生产者（producer）和消费者（consumer）对应的代码中修改。

这时候，我们就可以确保在RabbitMq重启之后test_queue队列不会丢失。另外，我们需要把我们的消息也要设为持久化——将MessageProperties的值设为PERSISTENT_TEXT_PLAIN。

```java
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

> #### 注意：消息持久化

> 将消息设为持久化并不能完全保证不会丢失。以上代码只是告诉了RabbitMq要把消息存到硬盘，但从RabbitMq收到消息到保存之间还是有一个很小的间隔时间。因为RabbitMq并不是所有的消息都使用fsync(2)——它有可能只是保存到缓存中，并不一定会写到硬盘中。并不能保证真正的持久化，但已经足够应付我们的简单工作队列。如果你一定要保证持久化，你可以使用**publisher confirms**。

##公平调度

你应该已经发现，它仍旧没有按照我们期望的那样进行分发。比如有两个工作者（workers），处理奇数消息的比较繁忙，处理偶数消息的比较轻松。然而RabbitMQ并不知道这些，它仍然一如既往的派发消息。

这时因为RabbitMQ只管分发进入队列的消息，不会关心有多少消费者（consumer）没有作出响应。它盲目的把第n-th条消息发给第n-th个消费者。

![](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

我们可以使用basicQos方法，并设置prefetchCount=1。这样是告诉RabbitMQ，再同一时刻，不要发送超过1条消息给一个工作者（worker），直到它已经处理了上一条消息并且作出了响应。这样，RabbitMQ就会把消息分发给下一个空闲的工作者（worker）。

    int prefetchCount = 1;
	channel.basicQos(prefetchCount);

> #### 关于队列大小

> 如果所有的工作者都处理繁忙状态，你的队列就会被填满。你需要留意这个问题，要么添加更多的工作者（workers），要么使用其他策略。

## 整合代码

NewTask.java的完整代码：

```java
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
```

([NewTask.java](hhttps://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/NewTask.java)源码)

我们的Worker.java：

```python
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
```

([Worker.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Worker.java) source)

使用消息确认和prefetchCount你就可以搭建起一个工作队列了。这些持久化的选项使得在RabbitMQ重启之后仍然能够恢复。

现在我们可以移步教程3学习如何发送相同的消息给多个消费者（consumers）。