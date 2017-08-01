# Parameters and Policies（参数和策略）

## 介绍

虽然RabbitMQ的大部分配置都位于配置文件中，但有些事情并不能很好地与配置文件配合使用：

1. 群集中的所有节点是否都需要相同
2. 它们在运行时是否可能会发生变化

RabbitMQ可以通过调用rabbitmqctl 或通过管理插件的HTTP API 来设置这些参数。有两种参数：vhost范围参数和全局参数。Vhost作用域的参数与虚拟主机相关联，并由组件名称，名称和值组成。全局参数不与特定虚拟机相关联，它们由名称和值组成。

策略是一种特殊的参数使用参数使用，它用于指定队列和交换组的可选参数，以及诸如联合和铲除之类的插件。策略是vhost-scope。

## 全局和每个vhost的参数

正如上面所说的，在这里我们开始学习全局和每个vhost的参数。vhost-scoped的一个例子就是federation upstream:它指向一个组件（federation-upstream），它具有标识它的名称，它绑定到一个虚拟主机（联合链接将瞄准该虚拟主机的某些资源），以及其值定义到上游代理的连接参数。可以设置、清除和列出Vhost范围的参数：

---------------------------

**rabbitmqctl**

	rabbitmqctl set_parameter {-p vhost} component_name name value
	rabbitmqctl clear_parameter {-p vhost} component_name name
	rabbitmqctl list_parameters {-p vhost}

**HTTP API**

	PUT /api/parameters/component_name/vhost/name
	DELETE /api/parameters/component_name/vhost/name
	GET /api/parameters

---------------------------

全局参数是另一种参数。全局参数的示例是集群的名称。可以设置、清除和列出全局参数：

---------------------------

**rabbitmqctl**

	rabbitmqctl set_global_parameter name value
	rabbitmqctl clear_global_parameter name
	rabbitmqctl list_global_parameters

**HTTP API**

	PUT /api/global-parameters/name
	DELETE /api/global-parameters/name
	GET /api/global-parameters

---------------------------

由于参数值是一个JSON文档，因此通常需要使用rabbitmqctl命令行来创建一个参数值。在Unix上，通常最简单的是使用单引号引用整个文档，并在其中使用双引号。在Windows上，您每次必须避免双引号。为此，我们给出了Unix和Windows的示例。

参数存在于RabbitMQ的虚拟主机（virtual hosts）、交换机（exchanges）、队列（queues）、绑定（bindings）、用户（users）和权限（permissions）定义的数据库中。参数可以通过管理插件的导出功能与其他对象定义一起导出。

Vhost范围的参数被用于联合和清除插件。全局参数被用于MQTT插件。