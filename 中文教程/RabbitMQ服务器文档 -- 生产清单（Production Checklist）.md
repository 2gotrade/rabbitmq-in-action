# 生产清单（Production Checklist）

## 介绍

数据服务（如RabbitMQ）通常具有许多可调参数。一些配置对开发有很大的意义，但并不适合生产。没有一个配置符合每个用例。因此，在进行生产之前对其配置进行评估是非常重要的。本指南旨在帮助您。

**虚拟主机，用户，权限**

## 虚拟主机（Virtual Hosts）

在单租户环境中，当您的RabbitMQ集群专用于为生产中的单个系统提供服务时，使用默认虚拟主机（/）是完全正常的。

在多租户环境中，为每个租户/环境使用单独的虚拟主机，例如project1_development， project1_production，project2_development，  project2_production等。

## 用户（Users）

对于生产环境，删除默认用户（guest）。默认用户只能从localhost连接，因为它具有众所周知的凭据。不要启用远程连接，请考虑使用具有管理权限和生成的密码的单独用户。

建议每个应用程序使用单独的用户。例如，如果您有移动应用程序，Web应用程序和数据聚合系统，则可以有3个单独的用户。这使得一些事情变得更容易：

1. 将客户端连接与应用程序相关联
2. 使用细粒度的权限
3. 凭证翻转（例如定期或在违约的情况下）

如果同一应用程序有许多实例，则在更好的安全性（每个实例具有一组凭据）和便利的配置（在一些或所有实例之间共享一组凭据）之间存在权衡。对于涉及许多客户端执行相同或相似功能并具有固定IP地址的IoT应用程序，使用[x509证书](http://www.rabbitmq.com/ssl.html)或[源IP地址范围](https://github.com/gotthardp/rabbitmq-auth-backend-ip-range)进行身份验证可能是有意义的。

**资源限制**

当跟踪消费者不时， RabbitMQ使用[资源驱动的闹钟](http://www.rabbitmq.com/alarms.html)来抑制发布者。在投入生产之前评估资源限制配置很重要。

## 内存

默认情况下，当RabbitMQ检测到使用超过40％的可用内存（由操作系统报告）时，RabbitMQ将不会接受任何新消息：  {vm_memory_high_watermark，0.4}。这是一个安全的默认值，即使主机是专用的RabbitMQ节点，修改此值时也应小心。

操作系统和文件系统使用系统内存来加速所有系统进程的操作。由于OS交换，由于没有足够的空闲系统内存将对系统性能产生不利影响，甚至可能导致RabbitMQ进程终止。

调整默认值vm_memory_high_watermark时有几个建议 ：

- 托管RabbitMQ的节点至少应该有128MB的内存可用。
- 推荐的vm_memory_high_watermark范围是  0.40到0.66
- 不推荐使用0.7 以上的值。操作系统和文件系统必须至少占用内存的30％，否则性能可能由于分页而严重恶化。

正如每次调优、基准测试和测量一样，需要为环境和工作负载找到最佳设置。

[了解有关RabbitMQ和系统内存的更多信息](http://www.rabbitmq.com/memory.html)

## 磁盘空间

当前的50MB disk_free_limit默认值对开发和教程非常有用。生产部署需要更大的安全余地。磁盘空间不足将导致节点出现故障，并可能导致数据丢失，因为所有磁盘写入都将失败。

为什么默认是50MB呢？开发环境有时使用真正的小分区来托管  /var/lib，这意味着节点在启动后立即进入资源报警状态。非常低的默认值确保了RabbitMQ为所有人开箱即用。至于生产部署，我们建议如下：

- {disk_free_limit，{mem_relative，1.0}}是最小推荐值，它转换为可用内存总量。例如，在专用于具有4GB系统内存的RabbitMQ的主机上，如果可用磁盘空间降至4GB以下，则所有发布者将被阻止，并且不会接受新的消息。排队队伍通常由消费者排出，出版前将被允许恢复。
- {disk_free_limit，{mem_relative，1.5}}是一个更安全的生产值。在具有4GB内存的RabbitMQ节点上，如果可用磁盘空间低于6GB，则所有新消息将被阻止，直到磁盘警报清除。如果RabbitMQ需要刷新4GB的磁盘数据，那么关机时有时可能会有这样的情况，RabbitMQ将有足够的磁盘空间重新启动。在这个具体的例子中，RabbitMQ将会启动并立即阻止所有发布商，因为2GB已经低于所需的6GB。
- {disk_free_limit，{mem_relative，2.0}}是最保守的生产价值，我们无法想到有什么更高的使用价值。如果您希望完全有信心，RabbitMQ具有所需的所有磁盘空间，在任何时候，这是使用价值。

## 打开文件句柄限制

操作系统限制同时打开文件句柄的最大数量，其中包括网络套接字。确保您的限制设置足够高以允许预期的并发连接数和队列数。

确保您的环境允许有效的RabbitMQ用户至少有50K的打开文件描述符，包括在开发环境中。

根据经验，将并发连接的第95百分位数乘以2，并添加总队列数来计算推荐的打开文件句柄限制。高达500K的值不足，不会消耗很多硬件资源，因此建议用于生产设置。请参阅[网络指南](http://www.rabbitmq.com/networking.html)了解更多信息。

## 安全注意事项

### 用户和权限

请参阅上面的vhosts，用户和凭据部分

### Erlang Cookie

在Linux和BSD系统上，有必要将Erlang Cookie 访问限制在运行RabbitMQ的用户和诸如rabbitmqctl之类的工具上。

### TLS

我们建议尽可能使用TLS连接，至少加密流量。同时建议对等验证（认证）。开发和质量检查环境可以使用自签名的TLS证书。当RabbitMQ和所有应用程序在受信任的网络上运行或使用VMware NSX等技术隔离时，自签名证书可以在生产环境中适用。

虽然RabbitMQ默认尝试提供安全的TLS配置（例如禁用SSLv3），但我们建议您评估是否启用TLS版本和密码套件。有关详细信息，请参阅我们的[TLS指南](http://www.rabbitmq.com/ssl.html)。

**网络配置**

生产环境可能需要网络配置调整。有关详细信息，请参阅[网络指南](http://www.rabbitmq.com/networking.html)。

### 自动恢复连接

一些客户端库，例如Java，.NET和Ruby，支持网络故障后的自动连接恢复。如果所使用的客户端提供此功能，建议使用它，而不是开发自己的恢复机制。

## 集群注意事项

### 群集大小

在确定群集大小时，考虑几个因素很重要：

1. 预期吞吐量
2. 预期复制（镜像数）
3. 数据位置

由于客户端可以连接到任何节点，所以RabbitMQ可能需要执行消息和内部操作的群集间路由。尝试使消费者和生产者连接到同一个节点，如果可能的话：这将减少节点间流量。同样有用的是使消费者连接到当前主持队列主节点的节点（可以使用HTTP API推断）。当考虑数据位置时，总体集群吞吐量可以达到非平凡的数量。

对于大多数环境，镜像到一半以上的群集节点就足够了。建议使用具有奇数节点（3，5，等等）的群集。

### 分区处理策略

在开始生产之前 选择[分区处理策略](http://www.rabbitmq.com/partitions.html)很重要。如有疑问，请使用自动对焦策略。

### 节点时间同步

RabbitMQ集群通常可以很好地运行，而不需要同步的参与服务器的时钟。然而，一些插件（例如管理UI）利用本地时间戳来进行度量处理，并且当节点的当前时间漂移分开时可能会显示不正确的统计信息。因此，建议服务器使用NTP或类似设备来确保时钟保持同步。

