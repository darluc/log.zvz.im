title: Gitlab 安装指南
date: 2016-04-02 00:37:15
tags: git
---
#Gitlab 安装指南

##Gitlab 简介

GitLab 是一个用于仓库管理系统的开源项目。使用 [Git](http://www.baike.com/sowiki/Git?prd=content_doc_search) 作为代码管理工具，并在此基础上搭建起来的 web 服务。 

1. Web框架使用 [Ruby on Rails](http://www.baike.com/sowiki/Ruby+on+Rails?prd=content_doc_search)。 
2. 基于 [MIT](http://www.baike.com/sowiki/MIT?prd=content_doc_search) 代码发布协议。

可以在 Github 上找到它的源码：https://github.com/gitlabhq/gitlabhq

如果你点击查看 [gitlabhq](https://github.com/gitlabhq) 这个用户，你可以看到更多的代码库。

<!--more-->  
![](http://ww4.sinaimg.cn/large/7327fe71gw1f29iby01w6j20jb0ihwfz.jpg)

截图中是以下三个项目：

1. **gitlabhq** 我们将要安装的 git server 端主程序；
2. **omnibus-gitlab** gitlab 一键式安装包，本文讲的是散装方式，不涉及此安装包；
3. **gitlab-shell** [gitolite](https://github.com/sitaramc/gitolite) 的替代程序，用于 git 仓库管理；

下面开始讲述安装过程。



##安装

> **说明**：本文以 centos 6.5 系统为例进行说明，由于是散装方式，故需要你会安装配置 nginx、mysql 等程序，本文将不再赘述。当然 git 的基本使用你也是要会的。以上如果有不会的建议收藏本文后，先去学习一下。

###第一步：环境准备###

1. 安装好系统环境，ruby & mysql/mariadb & redis & nginx（可选）。具体可以参考[参考文章一][1]

2. 创建用户 git

   ```shell
   $ su -
   $ adduser --system --shell /bin/bash --comment 'GitLab' --create-home --home-dir /home/git/ git
   ```

   **请不要擅自使用其它的用户名，这一点很重要。**

###第二步：下载 gitlab-shell 并修改配置

从 github 上下载 gitlab-shell 代码至 git 用户的 home 目录：

```shell
$ cd /home/git
$ git clone https://github.com/gitlabhq/gitlab-shell.git
$ cd gitlab-shell
```

然后切换至你所需要的版本，可以在 github 页面上找到对应的版本 tags：

![](http://ww3.sinaimg.cn/large/7327fe71gw1f29nzo8dboj20910dg74u.jpg)

```shell
$ git checkout v2.6.5
```

> 笔者安装的是 gitlab 7.x 的版本，需要跟 v2.6.5 的 gitlab-shell 配合，故以此版本号为例。

再修改一下它的配置文件：

```shell
$ cp config.yml.example config.yml
```

将 config.xml 中的 *gitlab_url* 修改为 gitlab 的访问域名，比如 http://git.zvz.im/。

###第三步：下载 gitlab 并修改配置

与上一步类似，我们先从 github 上下载 gitlab 代码至 git 用户的 home 目录：

```shell
$ cd /home/git
$ git clone https://github.com/gitlabhq/gitlabhq.git gitlab
```

然后切换至你所想要安装的版本：

```shell
$ git checkout 7-14-stable
```

> 由于8.x版本的 gitlab 需要 go 语言的支持，故笔者选择了安装了最新的 7.x 版本。

接下来，我们需要修改一下 gitlab 的配置文件：

```shell
# 复制配置文件
$ cp config/gitlab.yml.example config/gitlab.yml

# 修改配置文件中的域名(your_domain_name 为你给 gitlab 所分配的访问域名)
$ sed -i 's|localhost|your_domain_name|g' config/gitlab.yml

# 复制unicorn配置
$ cp config/unicorn.rb.example config/unicorn.rb
```

然后进行一些相关目录的设置：

```shell
# 设定log和tmp目录所有者和权限
$ chown -R git log/
$ chown -R git tmp/
$ chmod -R u+rwX log/
$ chmod -R u+rwX tmp/

# 创建gitlab-satellites目录
$ mkdir /home/git/gitlab-satellites

# 创建tmp/pids/和tmp/sockets/目录，确保gitlab有相应的权限
$ mkdir tmp/pids/
$ mkdir tmp/sockets/
$ chmod -R u+rwX tmp/pids/
$ chmod -R u+rwX tmp/sockets/

# 创建public/uploads目录
$ mkdir public/uploads
$ chmod -R u+rwX public/uploads
```

紧接着是数据库配置文件：

```shell
$ cp config/database.yml.mysql config/database.yml
```

> 如果你使用的是 postgreSql 数据库，那么你可以拷贝 config/database.yml.postgresql 这个文件。

然后编辑 _config/database.yml_ ，设置其中的数据库信息。

###第四步：安装 Gems 代码依赖

> Ruby 中的 gem 就是第三方依赖库，而 bundle 则是依赖包管理工具（类似于 Node.js 的 npm)。如果你的系统上没有安装 ruby 环境，请自行搜索解决如何安装。

开始安装依赖包：

```shell
$ cd /home/git/gitlab/
$ bundle install --deployment --without development test postgres puma aws
```

> 如果使用的是 postgeSql 数据库，则将上面指令中的 postgres 替换为 mysql 即可。
>
> **重要提示：**如果再这一步发生了一些安装错误或者提示网络连接超时的话，那么很有可能是因为 rubygems.org 访问受到了 GFW 防火墙的影响。要解决这个问题，只需要参考 https://ruby.taobao.org/ 上的说明，切换为淘宝的镜像源：对于 bundle 和 gems 的源最好都给换成国内的。
>
> 还有一种可能是你的系统上缺少某些库的开发文件，只要根据报错信息提示，安装相应 lib 的 devel 包即可。
###第五步：初始化数据库

```shell
$ cd /home/git/gitlab
$ bundle exec rake gitlab:setup RAILS_ENV=production
```

这里就是执行一些 gitlab 自身的初始化过程，主要是建立数据表并插入初始化数据。比如生成**默认的管理员账号**：

```
admin@local.host
5iveL!fe
```

###第六步：安装启动脚本

关于启动脚本，可以直接去这里 https://github.com/gitlabhq/gitlab-recipes/tree/master/init 找到适合你的系统的启动脚本文件即可，有可能里面某些命令的路径不对，只需要修改一下即可。

> 另外，我发现 6.x 版本 gitlab 的 script 目录中的执行文件在 6.x 以后被放到了 bin 目录下。启动脚本报错说某些文件不存在的时候，可以检查一下它的路径是不是错了。

到此为止，gitlab 就整个算是安装完成了。你可以执行以下命令来查看它的状态是否正常：

```shell
$ cd gitlab/
$ bundle exec rake gitlab:check RAILS_ENV=production
```


> 最后一个 nginx 的坑：如果安装完全没有问题而你又是使用 nginx 的时候，会发生用 http 协议push/pull git 库的时候报错；这个时候请检查一下你的 nginx 版本，如果太老了就请升级到最新版本吧，因为[老版本的 nginx 不支持文件分块（ chunked files ）的][2]。


最后给大家推荐一本 git 管理相关的书籍：[《Git版本控制管理(第2版)》](http://www.amazon.cn/gp/product/B00U42VM7Y/ref=as_li_tf_tl?ie=UTF8&camp=536&creative=3200&creativeASIN=B00U42VM7Y&linkCode=as2&tag=imzvz-23) <img src="http://ir-cn.amazon-adsystem.com/e/ir?t=imzvz-23&l=as2&o=28&a=B00U42VM7Y" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />






[1]: http://www.01happy.com/centos-6-5-install-gitlab/	"参考文章一"


[2​]: http://stackoverflow.com/questions/29898229/git-gitlab-push-rpc-failed-result-22-http-code-411	"stack overflow 上关于 gitlab 报错的问题"