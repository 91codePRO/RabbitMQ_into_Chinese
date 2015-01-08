>原文：[Hello World](http://www.rabbitmq.com/tutorials/tutorial-one-python.html)  
>状态：校对完成  
>翻译：Adam  
>校对：Ping

##介绍

RabbitMQ是一个消息代理。它的核心原理非常简单：接收和发送消息。你可以把它想像成一个邮局：你把信件放入邮箱，邮递员就会把信件投递到你的收件人处。在这个比喻中，RabbitMQ就是一个邮箱、邮局以及邮递员。

RabbitMQ和邮局的主要区别是，它不是用来处理纸张的，它是用来接收、存储和发送消息（message）这种二进制数据的。

一般提到RabbitMQ和消息，都会用到一些专有名词。

 - 生产(Producing)意思就是发送。发送消息的程序就是一个生产者(producer)。我们一般用"P"来表示:  
![](http://www.rabbitmq.com/img/tutorials/producer.png)

 - 队列(queue)就是邮箱的名称。消息通过你的应用程序和RabbitMQ进行传输，它们能够只存储在一个队列（queue）中。 队列（queue）没有任何限制，你要存储多少消息都可以——基本上是一个无限的缓冲。多个生产者（producers）能够把消息发送给同一个队列，同样，多个消费者（consumers）也能攻从一个队列（queue）中获取数据。队列可以化城这样（图上是队列的名称）：  
![](http://www.rabbitmq.com/img/tutorials/queue.png)

 - 消费（Consuming）和获取消息是一样的意思。一个消费者（consumer）就是一个等待获取消息的程序。我们把它画作"C"：  
![](http://www.rabbitmq.com/img/tutorials/consumer.png)

需要指出的是生产者、消费者、代理需不要待在同一个设备上；事实上大多数应用也确实不在会将他们放在一台机器上。

##Hello World!

**（使用pika 0.9.5 Python客户端）**

我们的“Hello world”不会很复杂——仅仅发送一个消息，然后获取它并输出到屏幕。这样以来我们需要两个程序，一个用作发送消息，另一个接受消息并打印消息内容

我们大体的设计是这样的：

![](http://www.rabbitmq.com/img/tutorials/python-one-overall.png)

生产者（producer）把消息发送到一个名为“hello”的队列中。消费者（consumer）从这个队列中获取消息。

>####RabbitMQ库

>RabbitMQ使用的是AMQP协议。要使用她你就必须需要一个使用同样协议的库。几乎所有的编程语言都有可选择的库。python也是一样，可以从以下几个库中选择：

>[py-amqplib](http://barryp.org/software/py-amqplib/)  
>[txAMQP](https://launchpad.net/txamqp)  
>[pika](http://github.com/pika/pika)  

>在这一系列教程中，我们打算使用pika。要安装pika，你可以使用pip这个包管理工具：

>     $ sudo pip install pika==0.9.5

>安装过程依赖于pip和git-core两个包，你需要先安装它们。

> - Ubuntu平台  
>        $ sudo apt-get install python-pip git-core

> - Debian平台  
>        $ sudo apt-get install python-setuptools git-core`  
>        $ sudo easy_install pip`

> - Windows平台 运行easy_install的安装程序setuptools即可，安装后运行以下命令  
>        > easy_install pip  
>        > pip install pika==0.9.5

###发送消息

![](http://www.rabbitmq.com/img/tutorials/sending.png)

我们第一个程序send.py会发送一个消息到队列中。首先要做的事情就是建立一个到RabbitMQ服务器的连接。

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
               'localhost'))
channel = connection.channel()
```

现在我们已经连接上服务器了，那么，在发送消息之前我们需要确认队列是存在的。如果我们把消息发送到一个不存在的队列，RabbitMQ会丢弃这条消息。我门先创建一个名为hello的队列，然后把消息发送到这个队列中。

```python
channel.queue_declare(queue='hello')
```

这时候我们就可以发送消息了，我们第一条消息只包含了 Hello World!字符串，我们打算把它发送到我们的hello队列。

在RabbitMQ中，消息是不能直接发送到队列，它需要发送到交换器（exchange）中。我们不打算在这里深入讨论它——你可以通过教程的第三部分了解更多。现在我们所需要了解的是如何使用默认的交换器（exchange），它使用一个空字符串来标识。交换器允许我们指定某条消息需要投递到哪个队列，routing_key参数必须指定为队列的名称：

```python
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print " [x] Sent 'Hello World!'"
```

在退出程序之前，我们需要确认网络缓冲已经被刷写、消息已经投递到RabbitMQ。完成这些事情（正确的关闭连接）是很简单的。

```python
connection.close()
```

> 发送不成功！

> 如果这是你第一次使用RabbitMQ，并且没有看到“Sent”消息出现在屏幕上，你可能会抓耳挠腮不知所以。这也许是因为没有足够的磁盘空间给代理使用所造成的（代理默认需要1Gb的空闲空间），所以它才会拒绝接收消息。查看一下代理的日志确定并且减少必要的限制。配置文件的文档会告诉你如何设置磁盘空间限制（disk_free_limit）。

###获取数据

![](http://www.rabbitmq.com/img/tutorials/sending.png)

我们的第二个程序receive.py，将会从队列中获取消息并打印消息。

这次我们还是先要连接到RabbitMQ服务器。连接服务器的代码和之前是一样的。

下一步也和之前一样，我们需要确认队列是存在的。使用queue_declare创建一个队列——我们可以运行这个命令很多次，但是只有一个队列会创建。

```python
channel.queue_declare(queue='hello')
```

你也许要问为什么重复声明了队列——我们已经在前面的代码中声明了它。如果我们确定了队列是已经存在的，那么我们可以不这么做。比如先运行send.py程序。可是我们并不确定哪个程序先运行，这种情况的话再程序中重复声明是好的做法。

>####列出所有队列

>你也许希望查看RabbitMQ由哪些队列、有多少消息在队列中。你可以使用rabbitmqctl工具（使用有权限的用户）：

>    ```bash
>    $ sudo rabbitmqctl list_queues
>    Listing queues ...
>    hello    0
>    ...done.
>    ```

>（在Windows中不需要sudo命令）

从队列中获取消息相对来说稍显复杂。需要为队列定义一个回调（callback）函数。当我们获取到消息的时候，Pika库就会调用这个回调（callback）函数。我们的这个回调函数将会但因消息的内容到屏幕上。

```python
def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
```

下一步，我们需要告诉RabbitMQ这个回调函数将会从hello队列中接收消息：

```python
channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)
```

要成功运行这些命令，我们必须保证队列是存在的，我们已经能够保证——我们之前已经使用创建了一个队列queue_declare。

no_ack参数稍后会介绍。

最后，我们输入一个无限循环来等待消息数据并在需要的时候运行回调函数。

```python
print ' [*] Waiting for messages. To exit press CTRL+C'
channel.start_consuming()
```

###整合

send.py的全部代码：

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print " [x] Sent 'Hello World!'"
connection.close()
```

([send.py 源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/send.py))

receive.py的全部代码：

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

print ' [*] Waiting for messages. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)

channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)

channel.start_consuming()
```

([receive.py source](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive.py))

现在就可以在终端中运行我们的程序了。首先，用send.py重续发送一条消息：

```bash
$ python send.py
[x] Sent 'Hello World!'
```

生产者（producer）程序send.py每次运行之后就会停止。现在我们就来接收消息：

```bash
$ python receive.py
[*] Waiting for messages. To exit press CTRL+C
[x] Received 'Hello World!'
```

成功了！我们已经通过RabbitMQ发送第一条消息。你也许已经注意到了，receive.py程序并没有退出。它一直在准备获取消息，你可以通过Ctrl-C来中止它。

试下在新的终端中再次运行send.py。

我们已经学会如何发送消息到一个已知队列中并接收消息。是时候移步到第二部分了，我们将会建立一个简单的工作队列（work queue）。