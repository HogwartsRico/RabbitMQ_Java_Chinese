## 远程过程调用（RPC）

#### （Python客户端 —— 使用 pika 0.9.8）

在[第二篇教程][4]中我们介绍了如何使用工作队列（work queue）在多个工作者（woker）中间分发耗时的任务。

可是如果我们需要将一个函数运行在远程计算机上并且等待从那儿获取结果时，该怎么办呢？这就是另外的故事了。这种模式通常被称为远程过程调用（Remote Procedure Call）或者RPC。

这篇教程中，我们会使用RabbitMQ来构建一个RPC系统：包含一个客户端和一个RPC服务器。现在的情况是，我们没有一个值得被分发的足够耗时的任务，所以接下来，我们会创建一个模拟RPC服务来返回斐波那契数列。

### 客户端接口

为了展示RPC服务如何使用，我们创建了一个简单的客户端类。它会暴露出一个名为“call”的方法用来发送一个RPC请求，并且在收到回应前保持阻塞。

```java
FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();   
String result = fibonacciRpc.call("4");
System.out.println( "fib(4) is " + result);
```

> ####关于RPC的注意事项：

> 尽管RPC在计算领域是一个常用模式，但它也经常被诟病。当一个问题被抛出的时候，程序员往往意识不到这到底是由本地调用还是由较慢的RPC调用引起的。同样的困惑还来自于系统的不可预测性和给调试工作带来的不必要的复杂性。跟软件精简不同的是，滥用RPC会导致不可维护的[面条代码][5].

> 考虑到这一点，牢记以下建议：

> 确保能够明确的搞清楚哪个函数是本地调用的，哪个函数是远程调用的。给你的系统编写文档。保持各个组件间的依赖明确。处理错误案例。明了客户端改如何处理RPC服务器的宕机和长时间无响应情况。

> 当对避免使用RPC有疑问的时候。如果可以的话，你应该尽量使用异步管道来代替RPC类的阻塞。结果被异步地推送到下一个计算场景。

### 回调队列

一般来说通过RabbitMQ来实现RPC是很容易的。一个客户端发送请求信息，服务器端将其应用到一个回复信息中。为了接收到回复信息，客户端需要在发送请求的时候同时发送一个回调队列（callback queue）的地址。我们可以使用默认的队列(这在java客户端是独有的),我们试试看：

```java
callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());

// ... then code to read a response message from the callback_queue ...
```

> ####消息属性

> AMQP协议给消息预定义了一系列的14个属性。大多数属性很少会用到，除了以下几个：

> * delivery_mode（投递模式）：将消息标记为持久的（值为2）或暂存的（除了2之外的其他任何值）。第二篇教程里接触过这个属性，记得吧？
* content_type（内容类型）:用来描述编码的mime-type。例如在实际使用中常常使用application/json来描述JOSN编码类型。
* reply_to（回复目标）：通常用来命名回调队列。
* correlation_id（关联标识）：用来将RPC的响应和请求关联起来。

我们需要导入一个新的类:

```import com.rabbitmq.client.AMQP.BasicProperties;
```

### 关联标识

上边介绍的方法中，我们建议给每一个RPC请求新建一个回调队列。这不是一个高效的做法，幸好这儿有一个更好的办法 —— 我们可以为每个客户端只建立一个独立的回调队列。

这就带来一个新问题，当此队列接收到一个响应的时候它无法辨别出这个响应是属于哪个请求的。** correlationId** 就是为了解决这个问题而来的。我们给每个请求设置一个独一无二的值。稍后，当我们从回调队列中接收到一个消息的时候，我们就可以查看这条属性从而将响应和请求匹配起来。如果我们接手到的消息的correlationId是未知的，那就直接销毁掉它，因为它不属于我们的任何一条请求。

你也许会问，为什么我们接收到未知消息的时候不抛出一个错误，而是要将它忽略掉？这是为了解决服务器端有可能发生的竞争情况。尽管可能性不大，但RPC服务器还是有可能在已将应答发送给我们但还未将确认消息发送给请求的情况下死掉。如果这种情况发生，RPC在重启后会重新处理请求。这就是为什么我们必须在客户端优雅的处理重复响应，同时RPC也需要尽可能保持幂等性。

### 总结

![](http://www.rabbitmq.com/img/tutorials/python-six.png)

我们的RPC如此工作:

* 当客户端启动的时候，它创建一个匿名独享的回调队列。
* 在RPC请求中，客户端发送带有两个属性的消息：一个是设置回调队列的 *replyTo* 属性，另一个是设置唯一值的 *correlationId* 属性。
* 将请求发送到一个 *rpc\_queue* 队列中。
* RPC工作者（又名：服务器）等待请求发送到这个队列中来。当请求出现的时候，它执行他的工作并且将带有执行结果的消息发送给replyTo字段指定的队列。
* 客户端等待回调队列里的数据。当有消息出现的时候，它会检查correlationId属性。如果此属性的值与请求匹配，将它返回给应用。

## 整合到一起
斐波那契数组任务:

```
private static int fib(int n) throws Exception {
    if (n == 0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
}
```
我们声明了我们的斐波那契方法.它假设只有可用的正整数的输入(不要期待输入比较大的数字也可以运行,因为它几乎可能是最慢的递归实现了).

RPCServer.java代码：

```java
private static final String RPC_QUEUE_NAME = "rpc_queue";

ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");

Connection connection = factory.newConnection();
Channel channel = connection.createChannel();

channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);

channel.basicQos(1);

QueueingConsumer consumer = new QueueingConsumer(channel);
channel.basicConsume(RPC_QUEUE_NAME, false, consumer);

System.out.println(" [x] Awaiting RPC requests");

while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();

    BasicProperties props = delivery.getProperties();
    BasicProperties replyProps = new BasicProperties
                                     .Builder()
                                     .correlationId(props.getCorrelationId())
                                     .build();

    String message = new String(delivery.getBody());
    int n = Integer.parseInt(message);

    System.out.println(" [.] fib(" + message + ")");
    String response = "" + fib(n);

    channel.basicPublish( "", props.getReplyTo(), replyProps, response.getBytes());

    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
}
```

服务器端代码相当简单：

* 像往常一样，我们建立连接，声明队列
* 我们为 basicConsume 声明了一个回调函数，这是RPC服务器端的核心。它执行实际的操作并且作出响应。
* 或许我们希望能在服务器上多开几个线程。为了能将负载平均地分摊到多个服务器，我们需要将 prefetchCount 设置好。

RPCClient 代码:

```java
private Connection connection;
private Channel channel;
private String requestQueueName = "rpc_queue";
private String replyQueueName;
private QueueingConsumer consumer;

public RPCClient() throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    connection = factory.newConnection();
    channel = connection.createChannel();

    replyQueueName = channel.queueDeclare().getQueue(); 
    consumer = new QueueingConsumer(channel);
    channel.basicConsume(replyQueueName, true, consumer);
}

public String call(String message) throws Exception {     
    String response = null;
    String corrId = java.util.UUID.randomUUID().toString();

    BasicProperties props = new BasicProperties
                                .Builder()
                                .correlationId(corrId)
                                .replyTo(replyQueueName)
                                .build();

    channel.basicPublish("", requestQueueName, props, message.getBytes());

    while (true) {
        QueueingConsumer.Delivery delivery = consumer.nextDelivery();
        if (delivery.getProperties().getCorrelationId().equals(corrId)) {
            response = new String(delivery.getBody());
            break;
        }
    }

    return response; 
}

public void close() throws Exception {
    connection.close();
}
```

客户端代码稍微有点难懂：

*  建立连接、通道并且为回复（replies）声明独享的回调队列。
*  我们订阅这个回调队列，以便接收RPC的响应。
*  接下来，我们定义我们的主要方法 call 方法。它执行真正的RPC请求。
*  接下来，我们将带有 replyTo 和 correlationId 属性的消息发布出去。
*  现在我们可以坐下来，等待正确的响应到来。
*  最后，我们将响应返回给用户。

客户端发送请求:

```java
RPCClient fibonacciRpc = new RPCClient();

System.out.println(" [x] Requesting fib(30)");   
String response = fibonacciRpc.call("30");
System.out.println(" [.] Got '" + response + "'");

fibonacciRpc.close();
```
编译代码:

```
$ javac -cp rabbitmq-client.jar RPCClient.java RPCServer.java
``

我们的RPC服务已经准备就绪了，现在启动服务器端：

```shell
$ java -cp $CP RPCServer
 [x] Awaiting RPC requests
```

运行客户端，请求一个fibonacci数字。

```shell
$ java -cp $CP RPCClient
 [x] Requesting fib(30)
```

此处呈现的设计并不是实现RPC服务的唯一方式，但是他有一些重要的优势：

* 如果RPC服务器运行的过慢的时候，你可以通过运行另外一个服务器端轻松扩展它。试试在控制台中运行第二个 rpc_server.py 。
* 在客户端，RPC请求只发送或接收一条消息。不需要像 queue_declare 这样的异步调用。所以RPC客户端的单个请求只需要一个网络往返。

我们的代码依旧非常简单，而且没有试图去解决一些复杂（但是重要）的问题，如：

* 当没有服务器运行时，客户端如何作出反映。
* 客户端是否需要实现类似RPC超时的东西。
* 如果服务器发生故障，并且抛出异常，应该被转发到客户端吗？
* 在处理前，防止混入无效的信息（例如检查边界）

> 如果你想做一些实验，你会发现[rabbitmq-management plugin][0]在观测队列方面是很有用处的。

（完整的[rpc_client.py][1] 和 [rpc_server.py][2]代码)


[0]:http://www.rabbitmq.com/plugins.html
[1]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCClient.java
[2]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCServer.java
[4]:http://www.rabbitmq.com/tutorials/tutorial-two-java.html
[5]:http://zh.wikipedia.org/wiki/%E9%9D%A2%E6%9D%A1%E5%BC%8F%E4%BB%A3%E7%A0%81