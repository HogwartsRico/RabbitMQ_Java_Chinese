##介绍

RabbitMQ是一个消息代理。它的核心原理非常简单：接收和发送消息。你可以把它想像成一个邮局：你把信件放入邮箱，邮递员就会把信件投递到你的收件人处。在这个比喻中，RabbitMQ就扮演着邮箱、邮局以及邮递员的角色。

RabbitMQ和邮局的主要区别是，它不是用来处理纸张的，它是用来接收、存储和发送消息（message）这种二进制数据的。

一般提到RabbitMQ和消息，都会用到一些专有名词。

 - 生产(Producing)意思就是发送。发送消息的程序就是一个生产者(producer)。我们一般用"P"来表示:  
![](http://www.rabbitmq.com/img/tutorials/producer.png)

 - 队列(queue)就是邮箱的名称。消息通过你的应用程序和RabbitMQ进行传输，它们能够只存储在一个队列（queue）中。 队列（queue）没有任何限制，你要存储多少消息都可以——基本上是一个无限的缓冲。多个生产者（producers）能够把消息发送给同一个队列，同样，多个消费者（consumers）也能够从同一个队列（queue）中获取数据。队列可以绘制成这样（图上是队列的名称）：  
![](http://www.rabbitmq.com/img/tutorials/queue.png)

 - 消费（Consuming）和获取消息是一样的意思。一个消费者（consumer）就是一个等待获取消息的程序。我们把它绘制为"C"：  
![](http://www.rabbitmq.com/img/tutorials/consumer.png)

需要指出的是生产者、消费者、代理需不要待在同一个设备上；事实上大多数应用也确实不在会将他们放在一台机器上。

##Hello World!

**（使用the Java Client）**

在教程的这部分,我们将要用Java写两个类;一个生产者(producer),它只发送一条消息,和一个消费者,它接受消息然后打印消息出来.我们将掩盖一些Java API中的细节,专注于让这个简单的Hello World程序跑起来.

我们的大致的设计是这样的：

![](http://www.rabbitmq.com/img/tutorials/python-one-overall.png)

生产者（producer）把消息发送到一个名为“hello”的队列中。消费者（consumer）从这个队列中获取消息。

>####The Java client library

>RabbitMQ可以有多种协议.这个教程使用AMQP 0-9-1协议,这个协议是一个开源的,多用途的消息协议.我们将使用RabbitMQ给出的java客户端来体验RebbitMQ.
>
>下载[rabbitmq的java客户端库](http://www.rabbitmq.com/java-client.html),解压然后获取我们要用的jar包

>     $ unzip rabbitmq-java-client-bin-*.zip
>     $ cp rabbitmq-java-client-bin-*/*.jar ./
>安装过程依赖于pip和git-core两个包，你需要先安装它们。
>(RabbitMQ的java客户端在maven的中央仓库也有,它的groupId是com.rabbitmq,artifactId是amqp-client)

现在我们有了rabbitmq的Java客户端库和它的依赖,我们可以开始敲代码了.
###发送消息

![](http://www.rabbitmq.com/img/tutorials/sending.png)

sender将会连接RabbitMQ,发送一个消息,然后退出

在Send.java里面,我们需要导入一些类;

```
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
```
建立这个类,以及给queue命名.

```
public class Send {
	private final static String QUEUE_NAME = "hello";
	
	public static void main(String[] argv){
		throws java.io.IOException {
		...
	}
}

```
然后我们创建连接:

```
 ConnectionFactory factory = new ConnectionFactory();
 factory.setHost("localhost");
 Connection connection = factory.newConnection();
 Channel channel = connection.createChannel();
```

这个连接封装了一个socket,同时处理好了消息协议的版本和认证.这里我们连接上了在本地(localhost)的一个中间人(broker),如果你想要连接别的主机上的中间人,只需要修改一下主机名字或者IP地址

下面,我们创建一个隧道(channel),这个隧道对象里面有我们需要的API.

要发送消息,我们必须先声明一个队列.

```
channel.queueDeclare(QUEUE_NAME , false , false , false , null);
String message = "Hello World";
channel.basicPublish("",QUEUE_NAME,null,message.getBytes());
System.out.println(" [x] Sent '"+message+"'");
```
声明一个队列是幂等操作 - 它将只在它不存在的时候被创建.消息的内容是一个字节数组

最后我们关闭隧道和连接;

```
    channel.close();
    connection.close();
```
[这里是完整的Send.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)

> 发送不成功！

> 如果这是你第一次使用RabbitMQ，并且没有看到“Sent”消息出现在屏幕上，你可能会抓耳挠腮不知所以。这也许是因为没有足够的磁盘空间给代理使用所造成的（代理默认需要1Gb的空闲空间），所以它才会拒绝接收消息。查看一下代理的日志确定并且减少必要的限制。配置文件文档会告诉你如何更改磁盘空间限制（disk_free_limit）。

###获取数据

![](https://www.rabbitmq.com/img/tutorials/receiving.png)

和发送者不一样,我们要让接收者保持运行状态来监听消息的接收.

Recv.java和Send.java的imports很相似:

```
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
```
这个额外的***DefaultConsumer***类实现了***Consumer***接口,我们要使用它来buffer接收到的消息

下面的代码和**Send.java**也很相似,都是要先打开连接得到隧道,声明队列:

```
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME,false,false,false,null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

```
因为消息的推送是异步的,所以我们在DefaultConsumer这个类中提供一个回调函数,这个回调函数里面buffer着传过来的消息.

```
    DefaultConsumer consumer = new DefaultConsumer(channel){

      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + message +"'");
      }
    };

    channel.basicConsume(QUEUE_NAME , true , consumer);

```

[这里是完整的Recv.java类的代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)

###整合
用rabbitmq-client.jar编译它们两个类

```
$ javac -cp rabbitmq-client.jar Send.java Recv.java
```

运行发送者:

```
$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Send
```
运行接收者:

```
$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Recv
```
接收者会在终端一直运行,等待消息从发送者传来.

我们已经学会如何发送消息到一个已知队列中并接收消息。是时候移步到第二部分了，我们将会建立一个简单的工作队列（work queue）。

