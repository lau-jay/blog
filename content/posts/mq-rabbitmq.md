+++
title = "MQ-RabbitMQ"
date = 2020-05-27T09:04:57+08:00
images = []
tags = []
categories = []
draft = false
+++

从一个简单的例子开始。

首先是生产者

```python
import sys
import pika

credentials = pika.PlainCredentials("guest", "guest")
conn_params = pika.ConnectionsParameters("localhost",  
    credentials= credentials)
conn_broker = pika.BlockingConnection(conn_params) # 到此为止,建立了到代理服务器的链接

channel = conn_broker.channel() # 获取信道
channel.exchange_declare(exchange="hello-exchange", 
         type="direct", passive=False, durable=True, 
         auto_delete=False) # 声明交换器

msg = sys.argv[1]
msg_props = pika.BasicProperties()
msg_props.content_type= “text/plain". # 创建纯文本消息

channel.basic_publish(body=msg, exchange="hello-exchange", 
                     properties=msg_props,
                     routing_key="hola") # 发布消息
conn_broker.close()
```

消费者:

```python
import sys
import pika

credentials = pika.PlainCredentials("guest", "guest")
conn_params = pika.ConnectionsParameters("localhost",
                   credentials= credentials)
conn_broker = pika.BlockingConnection(conn_params) # 到此为止,建立了到代理服务器的链接

channel = conn_broker.channel() # 获取信道
channel.exchange_declare(exchange="hello-exchange", 
               type="direct", passive=False, durable=True, 
               auto_delete=False) # 声明交换器

channel.queue_declare(queue="hello-queue") # 声明队列
channel.queue_bind(queue="hello-queue", exchange="hello-exchange",
               routing_key="hola") # 通过routing_key将队列和交换器绑定起来

def msg_consumer(channel, method, header, body):
    channel.basic_ack(delivery_tag=method.delivery_tag)
    if body.decode("utf8") == "quit": # 例子来源于rabbitmq实战,但是body在 python3中是byte类型
        channel.basic_cancel(consumer_tag="hello-consumer")
        channel.stop_consuming()
    else:
        print(body.decode("utf8"))
channel.basic_consume(queue=queue_name, 
        on_message_callback= msg_consumer, auto_ack=False)
channel.start_consuming()
```





## 生产者和消费者

生产者创建消息=》发布到代理服务器(RabbitMQ)

### 消息

消息分为: payload和label

payload是你要传输的内容,label描述了有效载荷.

AMQP用标签表述消息, rabbitmq根据标签把消息发送给感兴趣的接受放. 其方式是fire-and-forget

接受者得到payload, 标签在过程中不会发送

### 持久化消息

delivery mode 设置为2, 但是必须是持久化的交换器和持久化的队列, 三者加在一起才能真正具有持久化.

### 信道(channel)

信道事建立在“真实” TCP链接内的虚拟链接, 信道复用TCP链接,可以有无数个信道,但是会共用一个TCP链接.

## AMQP三要素

### 交换器(exchange)

生产者把消息发布到交换器上



交换器根据routing-key来决定消息投递到哪个队列

队列通过routing-key绑定到交换器.

交换器类型: topic, fanout, direct, headers

direct:  如果路由键匹配,就投递到对应的队列

fanout:  广播到所有绑定到该交换器的队列上

topic: 多个来源的消息到同一个队列上

### 持久化

durable设置为True

### 队列(queue)

消息到达队列,并被消费者接受

### 创建队列

queue.declare, 如果消费者在一条信道上订阅了某个队列,无法再声明队列了, 必须先取消订阅, 将 信道设置为“传输模式”.

queue.declare有两个参数:

* exclusive, 为true, 队列私有, 只能创建的使用,一个队列一个消费者

* auto-delete , 最后一个消费者取消订阅后, 队列自动移除.

交换器是topic情况下:queue.bind_queue通过指定队列名, 交换器名, 和绑定键名, 将队列绑定到交换器.

### 持久化

durable设置为True

### 绑定(bind_key)

绑定key决定了交换器将符合绑定key规则的消息都发送到这个队列里.

##  vhost虚拟主机

虚拟主机实现了自身的交换器,队列,和绑定, 并且带有权限控制

vhost只能通过命令行来创建删除

### 创建

```shell
# rabbitmqctl add_vhost[vhost_name]
```

### 删除

```shell
# rabbitmqctl delete_vhost[vhost_name]
```

### 显示

```shell
# rabbitmqctl list_vhosts
```

## 多个消费者订阅到同一队列,如何分发消息

rabbitmq拥有多个消费者的时候, 队列收到的消息将以(round-robin)的方式发送给消费者, 每个消息只会发送给一个消费者.



## ACK机制

消费者确认消息会basic.ack显式发送确认.或者auto-ack为true. 如果没有ack,链接断开,那么该消息会被发给下一个 消费者, 确保消息被正确处理.

如果没有返回ack,那么rabbitmq不会发送下一条消息给消费者.

## Reject机制

如果消费者容易出问题,但是消息又不想删除, 两种做法,第一断开链接,第二 basic.reject(版本大于2.0).

如果reject设置了requeue参数为true, 该消息会被发给下一个消费者.


