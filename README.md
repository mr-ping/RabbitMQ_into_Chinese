#RabbitMQ中文

欢迎各位RabbitMQ使用者们在此贡献RabbitMQ中文译文或有关RabbitMQ的有价值的原创文章。  
如转载文章至此，请事先征得原作者授权。

##使用说明：
###贡献方式：
您可以：

- 将希望别人翻译的或者自己打算翻译的文章提交到这里
    - 在`source`目录中创建Markdown格式的新文档，并在头部添加`>原文：原文档地址`以及`>状态：待翻译`
    - 添加新文档后，`pull request`到本项目，等待贡献者翻译、校对
- 翻译`source`目录内尚未翻译完成的文章（翻译前请先认领文章）
    - 认领目的：保证同一时间只有一人负责翻译，避免因个人习惯产生的风格冲突，避免重复劳动
    - 认领方式：凡状态为待翻译的文章都可认领。将文章状态改为`>状态：翻译中`后`pull request`到本项目
    - 开始翻译：您认领操作的`pull request`被合并进本项目后表示认领成功，可对其进行翻译
    - 完成翻译后：请在文章头部添加`>翻译：您的名字`，并将状态改为`>状态：待校对`;然后将其移动到`translated`目录
- 对`translated`目录内所有已完成翻译的文档进行校对（提出关于内容质量，技术术语等的修缮建议）
    - 完成校对后：请在文章头部添加`>校对：您的名字`，并将状态改为`>状态：校对完成`;之后将其移动到`docs`目录。

###项目结构

`source`暂时存储尚未翻译或者完成部分翻译的文章  
`translated`暂存翻译完成但尚未校对的文档  
`docs`目录存储已经完成校对的文章

###文档格式：

- 文档的文件名请使用英文
- 发布到此处的文档请统一使用[Markdown](http://zh.wikipedia.org/wiki/Markdown)格式。
- Markdown语法介绍可以[点击此处](http://wowubuntu.com/markdown/)查看。
- Markdown编辑器推荐：
    - 在线编辑器：[Markable](http://markable.in/)
    - 本地编辑器：[haroopress](http://pad.haroopress.com/user.html)

###Github流程：

 1. 创建并登录您的Github帐号
 2. 点击本项目右上角fork按钮，将本项目派生到您的帐号下
 3. 将您帐号下的派生clone到本地  
    `git clone your_repo_address`
 4. 在本地的项目中新建一个branch，在此分支中进行您的工作  
    `git branch new_branch_name`
 5. 将git切换到新建的branch  
    `git chechout new_branch_name`
 6. 将本项目地址添加为远端对象  
    `git remote add upstream https://github.com/mr-ping/RabbitMQ_into_Chinese.git`
 7. 获取远端更新  
    `git fetch upstream master`
 8. 将远端更新合并到本地  
    `git merge upstream/master`
 9. 修改后，推送到自己的远端库
    `git push origin HEAD:master`
 10. 使用自己GitHub中的Pull Request功能将自己的库合并至本项目原始库

其中1-6步为初始化操作只需执行一次  
7&8 为更新本地库的操作，可多次执行  
更新到GitHub时执行 9&10 步  

##文章列表：

 - [RabbitMQ on Wikipedia](http://zh.wikipedia.org/wiki/RabbitMQ) 
 - [AMQP 0-9-1 简介](./published/AMQP_0-9-1_Model_Explained.md)
 - [RabbitMQ教程](./published/tutorials_with_python)
   - [RabbitMQ教程 - 使用主题交换机](./published/tutorials_with_python/Topics.md)
 - [安装](./published/installation)
   - [支持RabbiMQ的平台](./published/installation/Platforms_supported_by_RabbitMQ.md)
   - [在Debian/Ubuntu中安装RabbitMQ](./published/installation/Installing_on_Debian_Ubuntu.md)

##活跃的贡献者（排名不分先后）：

[Ping](http://mr-ping.com)
