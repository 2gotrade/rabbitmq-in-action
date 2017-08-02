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

## 策略

- **策略为什么会存在**

在我们解释什么是策略以及如何使用它们之前，这个部分将有助于我们理解策略为什么会被引入到RabbitMQ中。

除了强制属性（例如持久或排他性）之外，RabbitMQ中的队列和交换具有可选参数（参数），有时称为 x参数。当客户声明队列（交换）并控制各种可选功能（如镜像或TTL）时，它们由客户端提供。

由客户端控制RabbitMQ支持的一些协议中的属性，通常情况下非常好用但可能不够灵活，例如：更新TTL值或镜像参数，需要应用程序更改，重新部署和队列重新声明（包括删除）。此外，我们没有办法控制队列和交换组的额外参数。为了解决上述的问题，引入了策略的概念。

策略通过名称（使用正则表达式模式）匹配一个或多个队列，并将其定义（可选参数的映射）附加到匹配队列的x参数。换句话说，可以使用策略一次配置多个队列的x参数，并通过更新策略定义一次更新它们。

在RabbitMQ的当前版本中，可以由策略控制的一组功能与可由客户端提供的参数控制的一组功能是不同的。

- **策略如何工作**

关键的策略属性有：

(1). 名称：它可以是任何东西，除了基于ASCII的名称没有空格是推荐的

(2). pattern：匹配一个或多个队列（交换）名称的正则表达式。可以使用任何正则表达式。

(3). 定义：将注入到匹配队列和交换的可选参数映射的一组键/值对（认为是一个JSON文档）

(4). 优先级：见下文

策略自动匹配交换和队列，并帮助确定它们的行为方式。每个交换或队列最多只能有一个策略匹配（请参见下面的组合策略定义），然后每个策略将一组键值对（策略定义）注入匹配的队列（交换）。

策略只能匹配队列，只能交换，或两者兼容。当创建策略时，使用apply-to标志来控制。

策略随时可以改变。当更新策略定义时，将重新应用对匹配交换机和队列的影响。通常它会立即发生，但对于非常繁忙的队列可能需要一点时间（比如说几秒钟）。

每次创建交换或队列时，都会匹配并应用策略，而不仅仅是在策略创建时。

策略可用于配置联合插件（federation plugin），镜像队列（mirrored queues），备用交换（alternate exchanges）， 死字（dead lettering）， 每个队列TTL（per-queue TTLs）和 最大队列长度（maximum queue length.）。

定义策略的示例如下所示：

**rabbitmqctl**	

	rabbitmqctl set_policy federate-me "^amq\." '{"federation-upstream-set":"all"}' --priority 1 --apply-to exchanges

**rabbitmqctl (Windows)**
	
	rabbitmqctl set_policy federate-me "^amq\." "{""federation-upstream-set"":""all""}" --priority 1 --apply-to exchanges

**HTTP API**
	
	PUT /api/policies/%2f/federate-me
                    {"pattern": "^amq\.",
                     "definition": {"federation-upstream-set":"all"},
                     "priority": 1,
                    "apply-to": "exchanges"}

**Web UI**	

	Navigate to Admin > Policies > Add / update a policy.
	Enter "federate-me" next to Name, "^amq\." next to Pattern, and select "Exchanges" next to Apply to.
	Enter "federation-upstream-set" = "all" in the first line next to Policy.
	Click Add policy.
	
在虚拟主机中“/”，所有这些名称以“amq”开头的交换机与“all”和“federation-upstream-set”键 匹配。

参数"pattern"就是用来匹配交换或队列名称的正则表达式。

如果有多个策略可以匹配到给定的交换或队列，则使用优先级最高的策略。

在“apply-to”的参数可以是“exchanges”，  “queues”或“all”。参数“apply-to” 和“priority”设置是可选的，在默认情况下，分别是“all”和“0”。

- **组合策略定义**

在某些情况下，我们可能需要对资源应用多个策略定义。例如，我们可能需要一个队列可以是联合和镜像的。在任何给定时间，最多一个策略将应用于资源，但是我们可以在该策略中应用多个定义。

组合策略定义将需要指定上游集，因此我们需要在定义中使用联合上游集合的关键字。另一方面，将一些队列定义为镜像，我们还需要为策略定义ha模式的关键字。由于策略定义只是一个JSON对象，所以我们可以在同一策略定义中同时使用这两个键。

以下是一个例子：

**rabbitmqctl**
	
	rabbitmqctl set_policy ha-fed "^hf\." '{"federation-upstream-set":"all","ha-mode":"all"}' \ --priority 1 --apply-to queues

**rabbitmqctl (Windows)**
	
	rabbitmqctl set_policy ha-fed "^hf\." "{""federation-upstream-set"":""all"", ""ha-mode"":""all""}" ^ --priority 1 --apply-to queues

**HTTP API**
	
	PUT /api/policies/%2f/ha-fed
	{"pattern": "^hf\.",
	 "definition": {"federation-upstream-set":"all", "ha-mode": "all"},
	 "priority": 1,
	 "apply-to": "queues"}

**Web UI**
	
	Navigate to Admin > Policies > Add / update a policy.
	Enter "ha-fed" next to Name, "^hf\." next to Pattern, and select "Queues" next to Apply to.
	Enter "federation-upstream-set" = "all" in the first line next to Policy.
	Enter "ha-mode" = "all" on the next line.
	Click Add policy.

通过这样做，所有队列匹配的模式“^hf\.”。将会有“federation-upstream-set” 和“ha-mode”定义。	
