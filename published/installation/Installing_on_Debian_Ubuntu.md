>原文：[Installing on Debian / Ubuntu](http://www.rabbitmq.com/install-debian.html)  
>状态：校对完成  
>翻译：[Ping](http://weibo.com/370321376)  
>校对：[Ping](http://weibo.com/370321376)  

##安装到 Debian / Ubuntu 系统中

###下载服服务器

| Description              |             Download            |     |
| ------------------------ | ------------------------------  | --- |
| Packaged as .deb for Debian-based Linux | [rabbitmq-server_3.4.3-1_all.deb][0] |([Signature][5])|

自Debian since 6.0 (squeeze) 和 Ubuntu 9.04 之后，rabbitmq-server就已经被内置其中了。然而这些被包含在内的版本往往过低。所以从我们网站上下载 .deb 文件来安装可以达到更好的效果。查看 [Debian安装包][1] 和 [Ubuntu安装包][2] 来确认适用于指定发行版的可用版本。

你可以使用`dpkg`来安装从上边下载来的安装包，也可以使用我们的APT库（下边介绍）。

所有的依赖都会被自动安装。

###运行RabbitMQ服务器

####自定义RabbitMQ环境变量

服务器会以默认方式启动。你可以[自定义RabbitMQ的环境][3]。也可以查看[如何配置组件][4]。

####开启服务器

当RabbitMQ安装完毕的时候服务器就会像后台程序一般运行起来了。作为一个管理员，可以像平常一样在Debian中使用以下命令启动和关闭服务  
`invoke-rc.d rabbitmq-server stop/start/etc.`

注意：服务器是使用`rabbitmq`这个系统用户来运行的。如果你改变了Mnesia数据库或者日志的位置，那么你必须确认这些文件属于此用户（同时更新系统变量）。

###我们的APT库

使用我们的APT库：

- 将以下的行添加到你的 /etc/apt/sources.list 文件中：

        deb http://www.rabbitmq.com/debian/ testing main

（请注意上边行中的 testing 指的是RabbitMQ发行状态，而不是指特定的Debian发行版。你可以将它使用在Debain的稳定版、测试版、非稳定版本中。对Ubuntu来说也是如此。我们之所以将版本描述为 testing 这个词是为了强调我们会频繁发布一些新的东西。）

- （可选的）为了避免未签名的错误信息，请使用apt-key(8)命令将我们的[公钥](http://www.rabbitmq.com/rabbitmq-signing-key-public.asc)添加到你的可信任密钥列表中：

        wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
        sudo apt-key add rabbitmq-signing-key-public.asc

- 运行

        apt-get update`

- 像平常一样安装软件包即可；例如

        sudo apt-get install rabbitmq-server

###控制系统限制

如果要调整系统限制（尤其是打开文件的句柄的数量）的话，可以通过编辑 /etc/default/rabbitmq-server 文件让服务启动的时候调用ulimit，例如：

    ulimit -n 1024

这将会设置此服务打开文件句柄的最大数量为1024个（这也是默认设置）。

##安全和端口

SELinux和类似机制或许会通过绑定端口的方式阻止RabbitMQ。当这种情况发生时，RabbitMQ会启动失败。请确认以下的端口是可以被打开的：

- 4369 (epmd), 25672 (Erlang distribution)
- 5672, 5671 (启用了 或者 未启用 TLS 的 AMQP 0-9-1)
- 15672 (如果管理插件被启用)
- 61613, 61614 (如果 STOMP 被启用)
- 1883, 8883 (如果 MQTT 被启用)

##默认用户访问

代理会建立一个用户名为“guest”密码为“guest”的用户。未经配置的客户端通常会使用这个凭据。默认情况下，这些凭据只能在链接到本机上的代理时使用，所以在链接到其他设备的代理之前，你需要做一些事情。

查看[访问控制](http://www.rabbitmq.com/access-control.html)，了解如何新建更多的用户，删除“guest”用户或者给“guest”用户赋予远程访问权限。

##管理代理

如果想要停止或者查看服务器状态等，你可以调用`rabbitmqctl`（在管理员权限下）。如果没有任何代理在运行，所有的rabbitmqctl命令都会给出“结点未找到”的报告。

- 调用`rabbitmqctl stop`来关闭服务器。  
- 调用`rabbitmqctl status`来查看代理是否运行。  

更多信息请查看 [rabbitmqctl 信息](http://www.rabbitmq.com/man/rabbitmqctl.1.man.html)

###日志

服务器的输出被发送到 RABBITMQ_LOG_BASE 目录的 RABBITMQ_NODENAME.log 文件中。一些额外的信息会被写入到 RABBITMQ_NODENAME-sasl.log 文件中。

代理总是会把新的信息添加到日志文件尾部，所以完整的日志历史可以被保存下来。

你可以使用 logrotate 程序来执行必要的循环和压缩工作，并且你还可以更改它。默认情况下，这个位于 /var/log/rabbitmq 文件中的脚本会每周执行一次。你可以查看 /etc/logrotate.d/rabbitmq-server 来对 logrotate 进行配置。


[0]:http://www.rabbitmq.com/releases/rabbitmq-server/v3.4.3/rabbitmq-server_3.4.3-1_all.deb
[1]:http://packages.qa.debian.org/r/rabbitmq-server.html
[2]:https://launchpad.net/ubuntu/+source/rabbitmq-server
[3]:http://www.rabbitmq.com/configure.html#customise-general-unix-environment
[4]:http://www.rabbitmq.com/configure.html#configuration-file
[5]:http://www.rabbitmq.com/releases/rabbitmq-server/v3.4.3/rabbitmq-server_3.4.3-1_all.deb.asc
