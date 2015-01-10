##Installing on Debian / Ubuntu
##安装到 Debian / Ubuntu 系统中

###Download the Server
###下载服服务器

| Description              |             Download            |     |
| ------------------------ | ------------------------------  | --- |
| Packaged as .deb for Debian-based Linux | [rabbitmq-server_3.4.3-1_all.deb][0] |([Signature][5])|

rabbitmq-server is included in Debian since 6.0 (squeeze) and in Ubuntu since 9.04. However, the versions included are often quite old. You will probably get better results installing the .deb from our website. Check the Debian package and Ubuntu package details for which version of the server is available for which versions of the distribution.  
自Debian since 6.0 (squeeze) 和 Ubuntu 9.04 之后，rabbitmq-server就已经被内置其中了。然而这些被包含在内的版本往往过低。所以从我们网站上下载 .deb 文件来安装可以达到更好的效果。查看 [Debian安装包][1] 和 [Ubuntu安装包][2] 来确认适用于指定发行版的可用版本。

You can either download it with the link above and install with dpkg, or use our APT repository (see below).  
你可以使用`dpkg`来安装从上边下载来的安装包，也可以使用我们的APT库（下边介绍）。

All dependencies should be met automatically.  
所有的依赖都会被自动安装。

###Run RabbitMQ Server
###运行RabbitMQ服务器

####Customise RabbitMQ Environment Variables
####定制RabbitMQ环境变量

The server should start using defaults. You can customise the RabbitMQ environment. Also see how to configure components.  
服务器会以默认方式启动。你可以[自定义RabbitMQ的环境][3]。也可以查看[如何配置组件][4]。

####Start the Server
####开启服务器

The server is started as a daemon by default when the RabbitMQ server package is installed.
As an administrator, start and stop the server as usual for Debian using invoke-rc.d rabbitmq-server stop/start/etc.  
当RabbitMQ安装完毕的时候服务器就会像后台程序一般运行起来了。作为一个管理员，可以像平常一样在Debian中使用以下命令启动和关闭服务  
`invoke-rc.d rabbitmq-server stop/start/etc.`

Note: The server is set up to run as system user rabbitmq. If you change the location of the Mnesia database or the logs, you must ensure the files are owned by this user (and also update the environment variables).  
注意：服务器是使用`rabbitmq`这个系统用户来运行的。如果你改变了Mnesia数据库或者日志的位置，那么你必须确认这些文件属于此用户（同时更新系统变量）。

###我们的APT库

使用我们的APT库：

- 将以下的行添加到你的 /etc/apt/sources.list 文件中：

        deb http://www.rabbitmq.com/debian/ testing main
(Please note that the word testing in this line refers to the state of our release of RabbitMQ, not any particular Debian distribution. You can use it with Debian stable, testing or unstable, as well as with Ubuntu. We describe the release as "testing" to emphasise that we release somewhat frequently.)  
（请注意上边行中的 testing 指的是RabbitMQ发行状态，而不是指特定的Debian发行版。你可以将它使用在Debain的稳定版、测试版、非稳定版本中。对Ubuntu来说也是如此。我们之所以将版本描述为 testing 这个词是为了强调我们会频繁发布一些新的东西。）

- （可选的）为了避免未签名的错误信息，请使用apt-key(8)命令将我们的[公钥](http://www.rabbitmq.com/rabbitmq-signing-key-public.asc)添加到你的可信任密钥列表中：

        wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
        sudo apt-key add rabbitmq-signing-key-public.asc

- 运行

        apt-get update`

- 像平常一样安装软件包即可；例如

        sudo apt-get install rabbitmq-server

###Controlling system limits
###控制系统限制

如果要调整系统限制（尤其是打开文件的句柄的数量）的话，可以通过编辑 /etc/default/rabbitmq-server 文件让服务启动的时候调用ulimit，例如：

    ulimit -n 1024

这将会设置此服务打开文件句柄的最大数量为1024个（这也是默认设置）。

##Security and Ports
##安全和端口

SELinux and similar mechanisms may prevent RabbitMQ from binding to a port. When that happens, RabbitMQ will fail to start. Make sure the following ports can be opened:  
SELinux和类似机制或许会通过绑定端口的方式阻止RabbitMQ。当这种情况发生时，RabbitMQ会启动失败。请确认以下的端口是可以被打开的：

- 4369 (epmd), 25672 (Erlang distribution)
- 5672, 5671 (启用了 或者 未启用 TLS 的 AMQP 0-9-1)
- 15672 (如果管理插件被启用)
- 61613, 61614 (如果 STOMP 被启用)
- 1883, 8883 (如果 MQTT 被启用)

##Default user access
##默认用户访问

The broker creates a user guest with password guest. Unconfigured clients will in general use these credentials. By default, these credentials can only be used when connecting to the broker as localhost so you will need to take action before connecting fromn any other machine.  
代理会建立一个用户名为`guest`密码为`guest`的用户。未经配置的客户端通常会使用这个凭据。默认情况下，这些凭据只能在链接到本机上的代理时使用，所以在链接到其他设备的代理之前，你需要做一些事情。

See the documentation on access control for information on how to create more users, delete the guest user, or allow remote access to the guest user.  
查看[访问控制](http://www.rabbitmq.com/access-control.html)，了解如何新建更多的用户，删除`guest`用户或者给`guest`用户赋予远程访问权限。

##Managing the Broker
##管理代理

To stop the server or check its status, etc., you can invoke rabbitmqctl (as an administrator). It should be available on the path. All rabbitmqctl commands will report the node absence if no broker is running.  
如果想要停止或者查看服务器状态等，你可以调用`rabbitmqctl`（在管理员权限下）。如果没有任何代理在运行，所有的rabbitmqctl命令都会报出“结点未找到”。

- Invoke rabbitmqctl stop to stop the server.  
- 调用`rabbitmqctl stop`来关闭服务器。  
- Invoke rabbitmqctl status to check whether it is running.  
- 调用`rabbitmqctl status`来查看代理是否运行。  

More info on rabbitmqctl.  
更多信息请查看[rabbitmqctl 信息](http://www.rabbitmq.com/man/rabbitmqctl.1.man.html)

###Logging
###日志

Output from the server is sent to a RABBITMQ_NODENAME.log file in the RABBITMQ_LOG_BASE directory. Additional log data is written to RABBITMQ_NODENAME-sasl.log.  
服务器的输出被发送到 RABBITMQ_LOG_BASE 目录的 RABBITMQ_NODENAME.log 文件中。一些额外的信息会被写入到 RABBITMQ_NODENAME-sasl.log 文件中。

The broker always appends to the log files, so a complete log history is retained.  
代理总是会把新的信息添加到日志文件尾部，所以完整的日志历史可以被保存下来。

You can use the logrotate program to do all necessary rotation and compression, and you can change it. By default, this script runs weekly on files located in default /var/log/rabbitmq directory. See /etc/logrotate.d/rabbitmq-server to configure logrotate.  
你可以使用 logrotate 程序来执行必要的循环和压缩工作，并且你还可以更改它。默认情况下，这个位于 /var/log/rabbitmq 文件中的脚本会每周执行一次。你可以查看 /etc/logrotate.d/rabbitmq-server 来配置 logrotate 程序。


[0]:http://www.rabbitmq.com/releases/rabbitmq-server/v3.4.3/rabbitmq-server_3.4.3-1_all.deb
[1]:http://packages.qa.debian.org/r/rabbitmq-server.html
[2]:https://launchpad.net/ubuntu/+source/rabbitmq-server
[3]:http://www.rabbitmq.com/configure.html#customise-general-unix-environment
[4]:http://www.rabbitmq.com/configure.html#configuration-file
[5]:http://www.rabbitmq.com/releases/rabbitmq-server/v3.4.3/rabbitmq-server_3.4.3-1_all.deb.asc
