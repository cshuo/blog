title: Openstack源码解读先导篇--AMQP
category: openstack
author: cshuo
date: 2017-02-20
tags: openstack

---
> Openstack遵循这样的设计原则：项目之间通过REST API进行通信；项目内部的各个服务进程是通过消息总线--AMQP来进行交互的。所以在进行Openstack源码解读之前，弄清楚组件内部服务的通信方式十分必要，否则很多代码看起来经常是一头雾水。

<!--more-->

目前有很多消息总线的开源实现，Openstack采用的是RabbitMQ，基于RabbitMQ，Openstack的oslo_messaging库实现了两种方式来完成项目内部服务进程的通信：
* 远程过程调用 (RPC, Remote Procedure Call)

  通过远程过程调用，一个服务进程(客户端)可以调用另一个远程服务进程(服务端)中的方法。Openstack采用这种方式主要是考虑到其如下优点：
  1) 实现了各个服务之间的松耦合；2) 实现了客户端和服务端之间的异步通信(客户端在发出远程调用消息时，并不要求服务端此时已经启动)；3) 随机的对远程调用进行负载均衡 (如果有多个服务端被开启运行，客户端发出的远程调用被透明的分配到第一个可用的服务端)。
* 事件通知

  某个服务进程可以把消息发布到消息总线上，其它的服务进程如果对该消息感兴趣，就可以从总线中拿到这个消息并做进一步的处理，处理的结果不会返回给消息发布者。这种通信方式，不仅可以在一个组件内部进行通信，也可以在不同组件之间发送通知；比如Ceilometer就是通过该方式获取其他项目发布的事件通知，从而实现计量和监控。

## AMQP详解
AMQP是一个异步消息传递所使用的应用层协议规范，主要包括了消息的导向、队列、路由、可靠性和安全性。下面以RabbitMQ为例对AMQP进行介绍，其架构如下图所示：

![](https://raw.githubusercontent.com/cshuo/bpic/master/amqp.png)

RabbitMQ模型中主要有以下主要概念：
1. Publisher: 消息发送者，主要是将消息发送给Exchange，并且指明消息的Rounting Key，以实现Exchange对消息的路由。

2. Consumer: 消息消费者，从消息队列中拿取消息。

3. Exchagne: 消息交换机，从发布方接收消息，并转发到相应的队列中去。

4. Queue：消息缓冲区，接受从交换机转发而来的消息，以便消费者使用。

5. Routing: 从发布者发送到交换机的消息需要携带一个Rounting Key, 对应的每个与交换机绑定的消息队列都有一个Binding Key, 通过二者的匹配，实现消息的路由。

AMQP模型里定义了多种类型的交换机，比较重要的有三种，实现了不同的消息路由方法：
* 直接式：该类交换机需要精确匹配消息的Routing Key与交换机的Binding Key, 匹配成功则发送到对应绑定的消息队里中。
* 主题式：一种消息发布-订阅模式，该类型的交换机可能会将消息路由到不同的消息队列中，Routing Key和Binding Key的匹配方式比直接式更灵活，支持通配符，"#"匹配零个或多个单词，“\*”匹配单个单词，单词之间是由“.”来分割的。比如“\*.hades.#”可以匹配”tom.hades”和“tom.hades.jack”，但是不能匹配"hades.jack"。
* 广播式：忽略Routing Key和Bingding Key，消息会被发送到所有绑定在交换机上的消息队列中去。

## AMQP实现RPC
Openstack各个组件内服务进程基于AMQP实现通信，实际上真正实现消息调用流程的是RPC机制。Openstack每个组件内的服务可能是RPC模型中的Invoker，也可能是Worker。Invoker是RPC模型中负责发送消息的组件，消息发送方式有两种：1) rpc.call和rpc.cast, 通过call方式调用，远程方法会被同步执行，Invoker会被阻塞直到结果返回，而cast方式，远程方法会被异步执行，结果不需返回。Worker负责从消息队列中取得消息，进行处理，根据情况再发送结果给Invoker。下面对rpc.call和rpc.cast的AMQP模型进行分别介绍。

### RPC Call
下图展示了rpc.call过程中消息在AMQP模型中的流向：
1. rpc.call操作被调用，产生一个Topic类型的Publisher，负责将消息发送给队列系统，其生命周期是消息的整个传递过程；与此同时，在发送消息之前，产生一个Direct类型的Consumer，用来等待响应消息。
2. 根据消息的Routing Key(比如"topic.host")，消息被路由到相应的消息队列中，然后，Topic类型的Consumer获取消息，交给对应的Worker进行进一步的任务处理。
3. 当任务处理完毕后，一个Direct类型的Publisher被实例化，负责将处理结果发送到队列系统。
4. 当消息被Exchage根据Routing Key(比如"msg_id")路由到对应的消息队列中去，第一步产生的Direct类型的Consumer获取消息，返回给Invoker。

![](https://raw.githubusercontent.com/cshuo/bpic/master/rpc_call.png)

### RPC Cast
rpc.cast不需要返回结果，所以它实际上就是rpc.call的前两步骤，消息在AMQP模型中的流向如下图所示：
![](https://raw.githubusercontent.com/cshuo/bpic/master/rpc_cast.png)
