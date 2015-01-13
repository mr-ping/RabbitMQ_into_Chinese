>原文：[Supported Platforms](http://www.rabbitmq.com/platforms.html)  
>状态：校对完成  
>翻译：[Ping](http://weibo.com/370321376)  
>校对：[Ping](http://weibo.com/370321376)  

##支持的平台

我们的目标是让RabbitMQ运行在尽可能广泛的平台之上。RabbitMQ有着运行在所有Erlang所支持的平台之上的潜力，从嵌入式系统到多核心集群还有基于云端的服务器。

以下的平台是Erlang语言所支持的，因此RabbitMQ可以运行其上：

- Solaris
- BSD
- Linux
- MacOSX
- TRU64
- Windows NT/2000/XP/Vista/Windows 7/Windows 8
- Windows Server 2003/2008/2012
- Windows 95, 98
- VxWorks

RabbitMQ的开源版本通常被部署在以下的平台上：

- Ubuntu和其他基于Debian的Linux发行版
- Fedora和其他基于RPM包管理方式的Linux发行版
- openSUSE和衍生的发行版（包括SLES和SLERT）
- Mac OS X
- Windows XP 和 后续版本

###Windows

RabbitMQ会运行在Windows XP及其之后的版本之上（Server 2003, Vista, Windows 7, Windows 8, Server 2008 and Server 2012）。尽管没有经过测试，但它应该也可以在Windows NT 以及 Windows 2000上良好的运行。

Windows Erlang 虚拟机能够以32位（所有可用版本）和64位（R15B往后）方式使用。将32位虚拟机运行在64位系统上的时候会有一些限制（如地址空间）存在。


###常见的 UNIX

尽管没有官方支持，但Erlang和RabbitMQ还是可以运行在大多数系统的POSIX层上，包括Solaris, FreeBSD, NetBSD, OpenBSD等等。

###虚拟平台

RabbitMQ可以运行在物理的或模拟的硬件中。这个特性同样允许将不支持的平台模拟成一个支持的平台来运行RabbitMQ。

如果要将RabbitMQ运行在EC2上，点击 [EC2 guide](http://www.rabbitmq.com/ec2.html) 查看更多细节。

##商业平台支持

 [RabbitMQ commercial documentation](https://www.vmware.com/support/pubs/vfabric-rabbitmq.html)上有一系列你可以付费购买的RabbitMQ商业支持平台。

##不支持的平台

一些平台是不被支持的，而且很可能永远不会：

- z/OS 和大多数的大型机
- 有内存限制的机器（小于16Mb）

如果你的平台不在此列或者你需要其他的帮助，请[联系我们](http://www.rabbitmq.com/contact.html)