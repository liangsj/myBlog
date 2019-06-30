---
title: Hello World
date: 2016-03-26 11:27:31
---
大家好，这是我第一篇博文。按照程序员的惯例。第一篇文章应该是叫hello world吧

## Quick Start
建立博客的目的主要还是用来自娱自乐。偶尔记录一下自己的生活。新学到的技术，或者对以往技术的感悟。如果有人看的话。希望能对向我一样在学习中的人有所帮助。
## platform 
aliyun centos 

因为工作一直是用的linux发行版是ubuntu，但是最便宜的aliyun是centos的。为了省点钱，只能在centos上多折腾一点。估计我们这一代程序员，从在学校开始，接触的都是ubuntu。centos应该不是很多人用。好在基本的都差不多。遇到不相同的部分，概念迁移+google一下。基本也能解决。

## tools

### nginx
nginx是web容器。我对其研究不深，暂时还是停留在只知道配置阶段。看了nginx官网的文档，我觉得它的反向代理很有用。对于服务器分流，减压。多服务器搭建应该很方便.

nginx install 
``` bash
$ sudo yum -y install nginx centos 仓库中安装
$ sudo systemctl start nginx       启动nginx
```
接下来输入你的aliyun IP地址就可以看到nginx的成功启动界面了。

nginx setting
nginx 的配置文件在 /etc/nginx/nginx.conf
``` bash
$ cd /etc/nginx/
$ sudo chmod +rw nginx.conf 将配置文件设置成当前用户可读写模式
$ sudo mv nginx.conf nginx.cong.bak  备份配置文件，防止修改错误还能找会来
$ sudo vim nginx.conf                用vim 打开文件
```
{%codeblock nginx.conf%}
server{
	root //标出根目录文件，就是一下hexo生产的静态文件
		index index.php index.html index.htm 设置文件的名字格式
}
{%endcodeblock%}
### hexo
hexo 是基于nodejs的静态博客生成工具。个人觉得还挺好用，主要还是操作简单
hexo install
``` bash
$ sudo yum -y install node 安装nodejs
$ sudo yum -y install npm  安装nodejs的npm仓库
$ npm install -g hexo-cli  安装hexo
```
hexo 操作十分简单
```bash
$ hexo init 初始化当初文件夹，生成博客工程
$ hexo g    生成静态文件
$ hexo server 打开hexo调试服务器。如果提示错误，先安装hexo server组件
```
更多可以查看 hexo(http://hexo.io)官网
