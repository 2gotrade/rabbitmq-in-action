# Virtual Hosts（虚拟主机）

## 介绍

RabbitMQ是多用户系统：连接、交换、队列、绑定、用户权限、策略和其他一些实体逻辑组的事物都属于虚拟主机。如果您熟悉[Apache中的虚拟主机](https://httpd.apache.org/docs/2.4/vhosts/) 或[Nginx中的服务器块](https://www.nginx.com/resources/wiki/start/topics/examples/server_blocks/)，这个想法是类似的。但是，有一个重要区别：Apache中的虚拟主机在配置文件中定义; RabbitMQ虚拟主机却是通过rabbitmqctl或HTTP API来创建的。

## 逻辑和物理分离

虚拟主机提供资源的逻辑分组和分离。物理资源的分离不是虚拟主机的目标，它应该被视为一个实现细节。

## 虚拟主机和客户端连接

虚拟主机有一个名字。当AMQP客户端连接到RabbitMQ时，它将指定要连接的虚拟机名称。如果身份验证成功并且所提供的用户名被授予对vhost的权限，则建立连接。

与虚拟主机的连接只能在该虚拟机中进行交换、排队、绑定等操作。只有当应用程序同时连接到两个虚拟机时，才可以使用不同vhosts中的队列和交换机进行“互连”。例如，应用程序可以从一个虚拟主机消耗，然后重新发布到另一个虚拟机。此方案可能涉及不同群集或相同群集（或单个节点）中的虚拟机。 RabbitMQ Shovel插件就是这种应用程序的一个例子。

## 虚拟主机和STOMP

像AMQP一样，STOMP包括[虚拟主机](https://stomp.github.io/stomp-specification-1.2.html#CONNECT_or_STOMP_Frame)的[概念](https://stomp.github.io/stomp-specification-1.2.html#CONNECT_or_STOMP_Frame)。有关详细信息，请参阅[STOMP指南](http://www.rabbitmq.com/stomp.html)。

## 虚拟主机和MQTT

与AMQP和STOMP不同，MQTT不具有虚拟主机的概念。默认情况下，MQTT连接使用单个RabbitMQ主机。有特定于MQTT的约定和功能使客户端可以连接到特定的虚拟机，而不需要任何客户端库修改。有关详细信息，请参阅[MQTT指南](http://www.rabbitmq.com/mqtt.html)。