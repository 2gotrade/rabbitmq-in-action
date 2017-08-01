# RabbitMQ Server在SUSE Lunix的安装实践

> 系统环境:
SUSE Linux Enterprise Server 12 x86_64

## 一、安装Erlang/OTP环境

- [官网的用户安装手册](http://erlang.org/doc/installation_guide/INSTALL.html#id61460)

 安装Erlang需要的编译环境gcc gcc-c++ unixODBC unixODBC-devel openssl openssl-devel 以及jdk1.6以上版本

- 方式一(使用Erlang版本管理工具编译安装)：

kerl可以实现把指定Erlang构建版本部署到远程服务器上, 这样在一个集群中, 我们可以在一个服务器上编译, 统一部署所有的集群节点的 Erlang 运行环境.

	./kerl deploy <[user@]host> [directory] [remote_directory]
例如：

	./kerl deploy test@192.168.8.100 /usr/local/erlang /usr/local/erlang

1. 下载kerl脚本工具
	
	curl -O https://raw.githubusercontent.com/kerl/kerl/master/kerl
	
或者

	wget https://raw.githubusercontent.com/kerl/kerl/master/kerl

2. ./kerl list releases 会出现Erlang的各个版本
3. ./kerl build 19.3 19.3 进行下载/编译
4. ./kerl install 19.3 /usr/local/erlang   (这里路径随你自己)进行安装
5. ./kerl list inst allations 检查安装好的Erlang
6. ./usr/local/erlang/activate进行激活
7.  在/etc/profile文件下添加环境变量
   
	export ERLANG_HOME=/usr/local/erlang
	export PATH=$PATH:$ERLANG_HOME/bin 

8.  执行 source /etc/profile
9.  erl -version验证是否已经安装Erlang

>  删除安装的Erlang方法

	  ./kerl delete build 19.3
	  ./kerl delete installation /usr/local/erlang
	  

- 方式二(手动下载源代码编译安装)：

[RabbitMQ的安装及配置](http://blog.csdn.net/kucha4/article/details/50766071)

1. 下载所需版本的源码
	
	wget http://erlang.org/download/otp_src_19.3.tar.gz

2. 解压otp_src_19.3.tar.gz源码包

	tar -zxvf otp_src_19.3.tar.gz

3. 进入源码包解压缩后的根目录
	
	cd otp_src_19.3

4. 执行./configure，相关参数的作用查看官网的用户安装手册
	
	./configure --prefix=/usr/local/erlang --enable-threads --enable-halfword-emulator --enable-smp-support --enable-kernel-poll --enable-sctp --enable-native-libs --enable-shared-zlib --enable-m64-build --enable-silent-rules

5.  执行make && make install

- 【Noted】编译可能出现如下错误：
	
	*********************************************************************
		********************** APPLICATIONS DISABLED **********************
		*********************************************************************
		crypto : No usable OpenSSL found
		odbc : ODBC library - link check failed
		ssh : No usable OpenSSL found
		ssl : No usable OpenSSL found
		*********************************************************************
		*********************************************************************
		********************** APPLICATIONS INFORMATION *******************
		********************************************************************* 
		     wx : wxWidgets not found, wx will NOT be usable 
		*********************************************************************

- 解决办法：
1).查看当前系统openssl的版本：
	
	openssl version -a
	
		OpenSSL 1.0.1i-fips 6 Aug 2014
		built on: 2014-08-19 12:07:37.000000000 +0000
		platform: linux-x86_64
		options:  bn(64,64) rc4(16x,int) des(idx,cisc,16,int) blowfish(idx)
		compiler: gcc -fPIC -DOPENSSL_PIC -DZLIB -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -m64 -DL_ENDIAN -DTERMIO -O3 -Wall -fmessage-length=0 -grecord-gcc-switches -fstack-protector -O2 -Wall -D_FORTIFY_SOURCE=2 -funwind-tables -fasynchronous-unwind-tables -g -std=gnu99 -Wa,--noexecstack -fomit-frame-pointer -DTERMIO -DPURIFY -DSSL_FORBID_ENULL -D_GNU_SOURCE -Wall -fstack-protector -Wa,--noexecstack  -DOPENSSL_IA32_SSE2 -DOPENSSL_BN_ASM_MONT -DOPENSSL_BN_ASM_MONT5 -DOPENSSL_BN_ASM_GF2m -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM -DMD5_ASM -DAES_ASM -DVPAES_ASM -DBSAES_ASM -DWHIRLPOOL_ASM -DGHASH_ASM
		OPENSSLDIR: "/etc/ssl"


>  经查阅资料知是由于erlang与当前系统的openssl的动态链接库不兼容及缺少openssl-devel的依赖

> [SuSe 11以编译安装的方式升级OpenSSH、OpenSSL及依赖问](http://www.mamicode.com/info-detail-1491869.html)

> [Linux/Unix下ODBC的安装、配置与编程](http://blog.chinaunix.net/uid-8795823-id-2013290.html)

2). 重新下载编译安装openssl和相关的依赖

由于openssl依赖的软件太多，所以在升级openssl时，不用卸载旧的版本。如果强制卸载可能导致系统不能正常运行
检查openssl的目录: 
	
	which openssl
	/usr/bin/openssl

在升级过程中将旧版的相关文件进行备份，在升级新版本后重新链接替换为新版本对应的文件目录
	
	whereis openssl
	openssl: /usr/bin/openssl /usr/include/openssl /usr/local/openssl /usr/share/man/man1/openssl.1ssl.gz

	ls /etc/ssl/
	certs  openssl.cnf  private  servercerts

备份上述文件，/usr/bin/X11/openssl为/usr/bin/openssl的软链接

	mkdir /home/ssl_bak
	mv /usr/bin/openssl /home/ssl_bak/
	mv /etc/ssl /home/ssl_bak/etc_ssl
	mv /usr/include/openssl /home/ssl_bak/include_openssl
       
升级openssl,安装必要的gcc、gcc-c++编译工具(如果已经安装，则跳过)及对应版本的libopenssl-devel、pam-devel、zlib-devel, [rpm包下载](https://pkgs.org/)
	
	## zypper in -y gcc gcc-c++
	wget http://ftp.gwdg.de/pub/opensuse/tumbleweed/repo/oss/suse/noarch/libopenssl-devel-1.0.2k-4.1.noarch.rpm
	wget http://ftp.gwdg.de/pub/opensuse/tumbleweed/repo/oss/suse/x86_64/pam-devel-1.3.0-5.1.x86_64.rpm
	wget http://ftp.gwdg.de/pub/opensuse/tumbleweed/repo/oss/suse/x86_64/zlib-devel-1.2.8-13.1.x86_64.rpm
	
	wget https://www.openssl.org/source/openssl-1.0.2k.tar.gz
	wget https://www.openssl.org/source/openssl-fips-2.0.13.tar.gz
	
	wget http://ftp.gwdg.de/pub/opensuse/tumbleweed/repo/oss/suse/x86_64/unixODBC-2.3.4-3.5.x86_64.rpm
	wget http://ftp.gwdg.de/pub/opensuse/tumbleweed/repo/oss/suse/x86_64/unixODBC-devel-2.3.4-3.5.x86_64.rpm
		
	rpm -ivh libopenssl-devel-1.0.2k-4.1.noarch.rpm --nodeps --force
	rpm -ivh pam-devel-1.3.0-5.1.x86_64.rpm --nodeps --force
	rpm -ivh zlib-devel-1.2.8-13.1.x86_64.rpm
	
	tar zxvf openssl-fips-2.0.13.tar.gz
	cd /usr/local/openssl-fips-2.0.13
	./config --prefix=/usr/local/openssl --openssldir=/etc/ssl shared
	make && make install
	
	tar zxvf openssl-1.0.2k.tar.gz
	cd /usr/local/openssl-1.0.2k
	./config --prefix=/usr/local/openssl --openssldir=/etc/ssl shared
	make && make install
	
	ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
	ln -s /usr/local/openssl/include/openssl /usr/include/openssl
	echo "/usr/local/openssl/lib" >> /etc/ld.so.conf
	ldconfig 
	###查看升级的openssl版本: openssl version
	
	rpm -ivh unixODBC-2.3.4-3.5.x86_64.rpm --nodeps --force
	rpm -ivh unixODBC-devel-2.3.4-3.5.x86_64.rpm

	
wxWidgets是一个跨平台的GUI库，可以忽略(不安装)

	wget http://ftp.gwdg.de/pub/opensuse/tumbleweed/repo/oss/suse/x86_64/wxWidgets-3_0-devel-3.0.2-12.1.x86_64.rpm
	rpm -ivh wxWidgets-3_0-devel-3.0.2-12.1.x86_64.rpm --nodeps --force

## 二、对RabbitMQ进行安装

1. 下载版本：版本号为3.6.9
	
	cd /usr/local
	wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.9/rabbitmq-server-generic-unix-3.6.9.tar.xz

2. 对于下载xz包进行解压(如果找不到xz命令，则需要下载xz压缩工具),这种下载的方式解压后直接可以使用，无需再编译安装
	
	xz -d rabbitmq-server-generic-unix-3.6.9.tar.xz
	tar -xvf rabbitmq-server-generic-unix-3.6.9.tar

3. 在/etc/profile文件下添加环境变量
	
	export RABBITMQ_HOME=/usr/local/rabbitmq_server-3.6.9
	export PATH=$PATH:$RABBITMQ_HOME/sbin

4. 执行 source /etc/profile

5. 启用MQ管理方式,使之可以通过网页方式管理MQ：
	
	rabbitmq-plugins enable rabbitmq_management

查看rabbitmq的进程：ps -ef | grep rabbitmq

到sbin/目录下运行./rabbitmq-server就可以启动rabbitmq服务

如果想以后台进程方式运行，那么可以运行rabbitmq-server -detached，此时RabbitMQ服务实例以后台线程方式运行了(忽略Warning: PID file not written; -detached was passed.)

如果需要停止正在后台运行的RabbitMQ服务实例, 可以运行rabbitmqctl stop

6. 用户管理

由于账号guest具有所有的操作权限，并且又是默认账号，出于安全因素的考虑，guest用户只能通过localhost登陆使用，并建议修改guest用户的密码以及新建其他账号管理使用rabbitmq(该功能是在3.3.0版本引入的)。

*  新增一个用户
	
	rabbitmqctl  add_user  Username  Password

*  删除一个用户
	
	rabbitmqctl  delete_user  Username

* 修改用户的密码
	
	rabbitmqctl  change_password  Username  Newpassword

* 查看当前用户列表
	
	rabbitmqctl  list_users

7. 用户角色

按照个人理解，用户角色可分为五类，超级管理员(administrator), 监控者(monitoring), 策略制定者(policymaker), 普通管理者(management)以及其他
设置用户角色的命令为：
	
	rabbitmqctl  set_user_tags  User  Tag

User为用户名， Tag为角色名(对应于上面的administrator，monitoring，policymaker，management，或其他自定义名称)。
也可以给同一用户设置多个角色，例如
	
	rabbitmqctl  set_user_tags  hncscwc  monitoring  policymaker

8. 用户权限

用户对exchange，queue的操作权限，包括配置权限，读写权限。配置权限会影响到exchange，queue的声明和删除。读写权限影响到从queue里取消息，向exchange发送消息以及queue和exchange的绑定(bind)操作

例如： 将queue绑定到某exchange上，需要具有queue的可写权限，以及exchange的可读权限；向exchange发送消息需要具有exchange的可写权限；从queue里取数据需要具有queue的可读权限。详细请参考官方文档中"How permissions work"部分。

相关命令为：

*  设置用户权限
	
	rabbitmqctl  set_permissions  -p  VHostPath  User  ConfP  WriteP  ReadP

* 查看(指定hostpath)所有用户的权限信息
	
	rabbitmqctl  list_permissions  [-p  VHostPath]

*  查看指定用户的权限信息
	
	rabbitmqctl  list_user_permissions  User

* 清除用户的权限信息
	
	rabbitmqctl  clear_permissions  [-p VHostPath]  User

9. 通过浏览器访问 http://192.168.1.48:15672/ 验证，在这里同样也可以对用户进行管理


## 三、RabbitMQ的配置和监控

1. 配置文件

主要参考[官方文档](http://www.rabbitmq.com/configure.html)
一般情况下，RabbitMQ的默认配置就足够了。如果希望特殊设置的话，有两个途径：
一般情况下，RabbitMQ的默认配置就足够了。如果希望特殊设置的话，有两个途径：
一个是环境变量的配置文件 rabbitmq-env.conf ；
一个是配置信息的配置文件 rabbitmq.config；

> 注意，这两个文件的位置分布特定的. 默认情况下，这些文件是没有创建的,但每个平台上期望的位置如下：

	Generic UNIX  $RABBITMQ_HOME/etc/rabbitmq/
	Debian  /etc/rabbitmq/
	RPM  /etc/rabbitmq/
	Mac OS X (Homebrew)  ${install_prefix}/etc/rabbitmq/, the Homebrew prefix is usually/usr/local
	Windows  %APPDATA%\RabbitMQ\

如果rabbitmq-env.conf不存在, 可在默认位置中手动创建. 它不能用于Windows系统.
如果rabbitmq.config不存在，可以手动创建它. 如果你修改了位置，可设置RABBITMQ_CONFIG_FILE 环境变量来指定. Erlang 运行时会自动在此变量值后添加.config扩展名，重启服务器后生效.
Windows 服务用户在删除配置文件后，需要重新安装服务．

* rabbitmq-env.conf

这个文件的位置是确定和不能改变的,文件的内容包括了RabbitMQ的一些环境变量，常用的有：

	#RABBITMQ_NODE_PORT=    //端口号
	#HOSTNAME=
	RABBITMQ_NODENAME=mq
	RABBITMQ_CONFIG_FILE=        //配置文件的路径
	RABBITMQ_MNESIA_BASE=/rabbitmq/data        //需要使用的MNESIA数据库的路径
	RABBITMQ_LOG_BASE=/rabbitmq/log        //log的路径
	RABBITMQ_PLUGINS_DIR=/rabbitmq/plugins    //插件的路径

具体参见[配置列表](http://www.rabbitmq.com/configure.html#define-environment-variables)

* rabbitmq.config

这是一个标准的erlang配置文件。它必须符合erlang配置文件的标准。
它既有默认的目录，也可以在rabbitmq-env.conf文件中配置。
详见[文件的内容](http://www.rabbitmq.com/configure.html#config-items)

2. RabbitMQ的监控

主要参考[官方文档](http://www.rabbitmq.com/management.html)
RabbitMQ提供了一个web的监控页面系统，这个系统是以Plugin的方式进行调用的。
首先，在rabbitmq-env.conf中配置好plugins目录的位置：RABBITMQ_CONFIG_FILE
将监控页面所需要的[plugin](http://www.rabbitmq.com/plugins.html#rabbitmq_manage)下载到plugins目录下，这些plugin包括：

	mochiweb
	webmachine
	rabbitmq_mochiweb
	amqp_client
	rabbitmq_management_agent
	rabbitmq_management
  
## 四、RabbitMQ的性能测试

1. 下载[测试工具](https://github.com/rabbitmq/rabbitmq-perf-test)

	wget https://github.com/rabbitmq/rabbitmq-perf-test/releases/download/v1.2.0/rabbitmq-perf-test-1.2.0-bin.tar.gz
	tar -zxvf rabbitmq-perf-test-1.2.0-bin.tar.gz
	cd rabbitmq-perf-test-1.2.0

* 查看脚本使有哪些参数可以使用

	bin/runjava com.rabbitmq.perf.PerfTest --help

	usage: <program>
	 -?,--help                         show usage
	 -A,--multiAckEvery <arg>          multi ack every
	 -a,--autoack                      auto ack
	 -b,--heartbeat <arg>              heartbeat interval
	 -C,--pmessages <arg>              producer message count
	 -c,--confirm <arg>                max unconfirmed publishes
	 -D,--cmessages <arg>              consumer message count
	 -d,--id <arg>                     test ID
	 -e,--exchange <arg>               exchange name
	 -f,--flag <arg>                   message flag
	 -h,--uri <arg>                    connection URI
	 -i,--interval <arg>               sampling interval in seconds
	 -K,--randomRoutingKey             use random routing key per message
	 -k,--routingKey <arg>             routing key
	 -M,--framemax <arg>               frame max
	 -m,--ptxsize <arg>                producer tx size
	 -n,--ctxsize <arg>                consumer tx size
	 -p,--predeclared                  allow use of predeclared objects
	 -Q,--globalQos <arg>              channel prefetch count
	 -q,--qos <arg>                    consumer prefetch count
	 -R,--consumerRate <arg>           consumer rate limit
	 -r,--rate <arg>                   producer rate limit
	 -s,--size <arg>                   message size in bytes
	 -t,--type <arg>                   exchange type
	 -u,--queue <arg>                  queue name
	 -X,--producerChannelCount <arg>   channels per producer
	 -x,--producers <arg>              producer count
	 -Y,--consumerChannelCount <arg>   channels per consumer
	 -y,--consumers <arg>              consumer count
	 -z,--time <arg>                   run duration in seconds (unlimited by default)

2. RabbitMQ 测试方案

方案1：启动一个生产者，一个消费者，生产者每次发送1条消息，每个消息的大小1KB(1000 bytes)，采样间隔时间为1s

	bin/runjava com.rabbitmq.perf.PerfTest -a -e test -k test -u test -x 1 -y 1 -i 1 -s 1000 -r 1

	id: test-092600-514, starting consumer #0
	id: test-092600-514, starting consumer #0, channel #0
	id: test-092600-514, starting producer #0
	id: test-092600-514, starting producer #0, channel #0
	id: test-092600-514, time: 1.349s, sent: 1.5 msg/s, received: 0.74 msg/s, min/avg/max latency: 8747/8747/8747 microseconds
	id: test-092600-514, time: 2.350s, sent: 1.00 msg/s, received: 1.00 msg/s, min/avg/max latency: 1322/1322/1322 microseconds
	id: test-092600-514, time: 3.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1576/1576/1576 microseconds
	id: test-092600-514, time: 4.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1764/1764/1764 microseconds
	id: test-092600-514, time: 5.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1321/1321/1321 microseconds
	id: test-092600-514, time: 6.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1257/1257/1257 microseconds
	id: test-092600-514, time: 7.350s, sent: 1.0 msg/s, received: 2.0 msg/s, min/avg/max latency: 936/1032/1127 microseconds
	id: test-092600-514, time: 8.350s, sent: 1.0 msg/s, received: 0 msg/s
	id: test-092600-514, time: 9.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 992/992/992 microseconds
	id: test-092600-514, time: 10.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1016/1016/1016 microseconds
	id: test-092600-514, time: 11.350s, sent: 1.0 msg/s, received: 2.0 msg/s, min/avg/max latency: 974/1145/1316 microseconds
	id: test-092600-514, time: 12.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 950/950/950 microseconds
	id: test-092600-514, time: 13.350s, sent: 1.0 msg/s, received: 0 msg/s
	id: test-092600-514, time: 14.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1003/1003/1003 microseconds
	id: test-092600-514, time: 15.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 991/991/991 microseconds
	id: test-092600-514, time: 16.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 937/937/937 microseconds
	id: test-092600-514, time: 17.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1051/1051/1051 microseconds
	id: test-092600-514, time: 18.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1047/1047/1047 microseconds
	id: test-092600-514, time: 19.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 977/977/977 microseconds
	id: test-092600-514, time: 20.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1018/1018/1018 microseconds
	id: test-092600-514, time: 21.350s, sent: 1.0 msg/s, received: 2.0 msg/s, min/avg/max latency: 910/971/1031 microseconds
	id: test-092600-514, time: 22.350s, sent: 1.0 msg/s, received: 0 msg/s
	id: test-092600-514, time: 23.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 983/983/983 microseconds
	id: test-092600-514, time: 24.350s, sent: 1.0 msg/s, received: 2.0 msg/s, min/avg/max latency: 921/946/972 microseconds
	id: test-092600-514, time: 25.350s, sent: 1.0 msg/s, received: 0 msg/s
	id: test-092600-514, time: 26.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1326/1326/1326 microseconds
	id: test-092600-514, time: 27.350s, sent: 1.0 msg/s, received: 2.0 msg/s, min/avg/max latency: 992/1012/1032 microseconds
	id: test-092600-514, time: 28.350s, sent: 1.0 msg/s, received: 0 msg/s
	id: test-092600-514, time: 29.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1039/1039/1039 microseconds
	id: test-092600-514, time: 30.350s, sent: 1.0 msg/s, received: 1.0 msg/s, min/avg/max latency: 1019/1019/1019 microseconds

由此可以推断出生产者每次发送1条消息，到消费者接收这条消息的平均延时是1毫秒

方案2：启动一个生产者，一个消费者，每个消息的大小1KB(1000 bytes)，采样间隔时间为1s

	bin/runjava com.rabbitmq.perf.PerfTest -a -e test -k test -u test -x 1 -y 1 -i 1 -s 1000

	id: test-223925-454, starting consumer #0
	id: test-223925-454, starting consumer #0, channel #0
	id: test-223925-454, starting producer #0
	id: test-223925-454, starting producer #0, channel #0
	id: test-223925-454, time: 1.000s, sent: 5335 msg/s, received: 3675 msg/s, min/avg/max latency: 5732/110237/168221 microseconds
	id: test-223925-454, time: 2.000s, sent: 18006 msg/s, received: 13412 msg/s, min/avg/max latency: 122550/219377/418626 microseconds
	id: test-223925-454, time: 3.000s, sent: 15326 msg/s, received: 15787 msg/s, min/avg/max latency: 281081/382660/477743 microseconds
	id: test-223925-454, time: 4.000s, sent: 17002 msg/s, received: 17741 msg/s, min/avg/max latency: 239863/309415/375502 microseconds
	id: test-223925-454, time: 5.000s, sent: 18305 msg/s, received: 18185 msg/s, min/avg/max latency: 239745/303885/370773 microseconds
	id: test-223925-454, time: 6.000s, sent: 18181 msg/s, received: 18089 msg/s, min/avg/max latency: 238403/303851/356413 microseconds
	id: test-223925-454, time: 7.000s, sent: 18305 msg/s, received: 18168 msg/s, min/avg/max latency: 249049/303426/352494 microseconds
	id: test-223925-454, time: 8.000s, sent: 17685 msg/s, received: 18116 msg/s, min/avg/max latency: 239857/299901/353704 microseconds
	id: test-223925-454, time: 9.000s, sent: 19422 msg/s, received: 18247 msg/s, min/avg/max latency: 239847/291086/345903 microseconds
	id: test-223925-454, time: 10.000s, sent: 16940 msg/s, received: 18546 msg/s, min/avg/max latency: 228982/298273/361520 microseconds
	id: test-223925-454, time: 11.000s, sent: 18864 msg/s, received: 17651 msg/s, min/avg/max latency: 254675/313990/381321 microseconds
	id: test-223925-454, time: 12.000s, sent: 17932 msg/s, received: 18110 msg/s, min/avg/max latency: 233456/297734/353409 microseconds
	id: test-223925-454, time: 13.000s, sent: 18381 msg/s, received: 18514 msg/s, min/avg/max latency: 237936/302431/356798 microseconds
	id: test-223925-454, time: 14.000s, sent: 17290 msg/s, received: 17600 msg/s, min/avg/max latency: 246758/315346/384957 microseconds
	id: test-223925-454, time: 15.000s, sent: 18128 msg/s, received: 18369 msg/s, min/avg/max latency: 237952/298335/360683 microseconds
	id: test-223925-454, time: 16.000s, sent: 18181 msg/s, received: 17794 msg/s, min/avg/max latency: 253386/310752/367168 microseconds
	id: test-223925-454, time: 17.000s, sent: 17684 msg/s, received: 17610 msg/s, min/avg/max latency: 243739/315561/382346 microseconds
	id: test-223925-454, time: 18.000s, sent: 19100 msg/s, received: 18568 msg/s, min/avg/max latency: 227744/290366/359178 microseconds
	id: test-223925-454, time: 19.000s, sent: 17324 msg/s, received: 17988 msg/s, min/avg/max latency: 248381/306870/364657 microseconds
	id: test-223925-454, time: 20.000s, sent: 18306 msg/s, received: 18328 msg/s, min/avg/max latency: 235593/294957/351625 microseconds
	id: test-223925-454, time: 21.000s, sent: 17498 msg/s, received: 18064 msg/s, min/avg/max latency: 247087/305115/364772 microseconds
	id: test-223925-454, time: 22.000s, sent: 18305 msg/s, received: 17874 msg/s, min/avg/max latency: 236146/301135/363878 microseconds
	id: test-223925-454, time: 23.000s, sent: 17916 msg/s, received: 17593 msg/s, min/avg/max latency: 248759/306640/363159 microseconds
	id: test-223925-454, time: 24.000s, sent: 18570 msg/s, received: 18487 msg/s, min/avg/max latency: 238732/301628/355898 microseconds
	id: test-223925-454, time: 25.000s, sent: 17995 msg/s, received: 18243 msg/s, min/avg/max latency: 240277/297902/345038 microseconds
	id: test-223925-454, time: 26.000s, sent: 19112 msg/s, received: 18307 msg/s, min/avg/max latency: 223122/294241/343911 microseconds
	id: test-223925-454, time: 27.000s, sent: 17746 msg/s, received: 18319 msg/s, min/avg/max latency: 241441/300598/353247 microseconds
	id: test-223925-454, time: 28.000s, sent: 18119 msg/s, received: 17753 msg/s, min/avg/max latency: 247164/311623/373695 microseconds
	id: test-223925-454, time: 29.000s, sent: 17561 msg/s, received: 17930 msg/s, min/avg/max latency: 233158/309824/377001 microseconds
	id: test-223925-454, time: 30.000s, sent: 18305 msg/s, received: 19088 msg/s, min/avg/max latency: 213279/273376/325914 microseconds

由于第一个1s开始时,队列中没有堆积消息,所以由生产者发送消息到消费者接收消息的延时较低;当队列产生堆积消息、生产和消费速率趋于稳定后,由生产者发送消息到消费者接收消息的延时也趋于稳定

如上输出的采样统计数据结果显示：

第一个1s内生产者发送了5335条消息，消费者接收了3675条消息，平均延时110237微秒(0.110237秒)，最大延时168221微秒(0.168221秒)，因此产生了5335-3675=1660条消息堆积

第二个1s内生产者发送了18006条消息，消费者接收了13412条消息，平均延时329422微秒(0.219377秒)，最大延时477757微秒(0.418626秒)，因此产生了18006+1600-13412=6254条消息堆积

由此可以推断出该情况下，MQ的发送和接收的速率是15000--19000 msg/s，平均延时300毫秒


## 五、RabbitMQ集群安装配置

1. 集群环境

	rabbitmq-node1 192.168.1.48
	rabbitmq-node2 192.168.1.49
	rabbitmq-node3 192.168.1.50

2. 修改所有集群机器的host文件

	192.168.1.48	rabbitmq-node1
	192.168.1.49	rabbitmq-node2
	192.168.1.50	rabbitmq-node3

保证两台机器都能够相互ping通

3. 安装Erlang和RabbitMQ

4. RabbitMQ集群配置

(1).  设置每个节点的Cookie

Rabbitmq的集群是依赖于erlang的集群来工作的，所以必须先构建起erlang的集群环境。
Erlang的集群中各节点是通过一个magic cookie来实现的，这个cookie存放在.erlang.cookie中，文件是400的权限。
所以必须保证各节点cookie保持一致，否则节点之间就无法通信。

> 官方在介绍集群的文档中提到过.erlang.cookie一般会存在这两个地址：第一个是$home/.erlang.cookie；第二个地方就是/var/lib/rabbitmq/.erlang.cookie。
* 如果我们使用解压缩方式安装部署的rabbitmq，那么这个文件会在${home}目录下，也就是$home/.erlang.cookie。
* 如果我们使用rpm等安装包方式进行安装的，那么这个文件会在/var/lib/rabbitmq目录下。

> 我们可以通过rabbitmq的启动日志查看其home目录是哪里，就可以知道.erlang.cookie存放在哪里，以及mnesia数据库信息存在哪里。

	=INFO REPORT==== 26-May-2017::17:01:30 ===
	Starting RabbitMQ 3.6.9 on Erlang 19.3
	Copyright (C) 2007-2016 Pivotal Software, Inc.
	Licensed under the MPL.  See http://www.rabbitmq.com/
	
	=INFO REPORT==== 26-May-2017::17:01:30 ===
	node           : rabbit@rabbitmq-node1
	home dir       : /home/xml
	config file(s) : /usr/local/rabbitmq/etc/rabbitmq/rabbitmq.config (not found)
	cookie hash    : w7r7zTnLUOM8X8AIqOwfaw==
	log            : /usr/local/rabbitmq/var/log/rabbitmq/rabbit@rabbitmq-node1.log
	sasl log       : /usr/local/rabbitmq/var/log/rabbitmq/rabbit@rabbitmq-node1-sasl.log
	database dir   : /usr/local/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@rabbitmq-node1
