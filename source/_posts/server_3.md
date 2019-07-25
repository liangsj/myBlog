---
title: 服务的搭建及其演变(3) - 负载平衡和简单的注册中心
date: 2019-07-25 00:00:00
categories:
 - 编程
tags: 
    - dockerfile
    - nginx
    - confd
---
# 服务的搭建及其演变 - 负载平衡和简单的注册中心
前两章中，我们通过添加缓存、将单点改造成分布式、并添加自动主从切换机制建立了一套高可用性的系统。存储层的瓶颈我们基本已经解决。但，随着业务的发展，流量的增加，我们的webserver模块单点劣势在逐步暴露，随着业务的迭代，计算会更加的复杂。websever 会成为架构中新的瓶颈。顺着如上的思路，我们需要优雅的解决webserver单点问题。
## 本节工具
- nginx : 高性能的http和反向代理的web服务器,本章用于做负载均衡
- confd : 配置管理工具，可以连接多种存储(ectd/redis等),通过模板管理服务器上配置
- dockerfile : docker 容器构建文件，通过编写dockerfile可以建造出服务需求的docker image
## webserver 的负载均衡
- 多webapp的搭建
- nginx 的搭建
- nginx 配置 
想将之前的web服务进行扩容，又之前的单webapp 扩到 三个，依旧用docker 来模拟分布式环境
```shell
webapp_1:
        image: golang
        container_name: webapp_1
        ports:
            - 8081:80
        networks:
            - front
        volumes:
            - ./go/webapp:/go/src/webapp
            - ./sh:/sh
        command: sh /sh/start.sh 
   webapp_2:
        image: golang
        container_name: webapp_2
        ports:
            - 8082:80
        networks:
            - front
        volumes:
            - ./go/webapp:/go/src/webapp
            - ./sh:/sh
        command: sh /sh/start.sh  
```
如上，现在有3个webapp，现在需要将这3个web服务器进行组合。并将其至于nginx后面，nginx作为唯一入口，并做负载均衡的工作<\t>
启动一个 nginx docker 
```shell
nginx:
        image: nginx
        container_name: nginx
        volumes:
            - ./conf/nginx/conf.d:/etc/nginx/conf.d
        ports:
            - 8220:8220
            - 8280:80
        networks:
            - front
```
其中，把负责均衡配置载入nginx中 
```shell
upstream webapp{              # 进行服务的三个webapp
  server webapp:80;
  server webapp_1:80;
  server webapp_2:80;
}

server {
    listen       8220;                #服务端口
    server_name  webapp;

    access_log  /var/log/nginx/host.access.log  main;

    location / {
        proxy_pass  http://webapp;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
 }

```
现在有三个可以统一协调工作的webapp了，单点问题被我们完美解决了，现架构如下图
{% img /image/3_webapp_stuct_1.png %}

我们多访问几次,通过nginx 的access_log我们可以看出，下游的webserver都被访问到了
{% img /image/3_webapp_ret_1.png %}

## 简单的注册中心搭建
大流量服务一般机器数是成百上千的。如果按照如上的方法进行配置工作量是巨大的，加上手工配置容易出错，存在这稳定性的风险。服务发现和服务注册是微服务架构的z重要组成部分,通过注册中心，可用实现快速的缩扩容，增加集群稳定性和灵活性。基本上每个公司都都大同小异的实现了这一组件，业界也有较为成熟的开源方案。我在这里实现一个较为简单，但是在不是很复杂的系统中也可用使用的服务注册中心的方案.
首先，要实现如下两步：
- webserver 的自动向注册中心注册
- 负载均衡组件的自动更新

{% img /images/3_webapp_struct_2.png %}

### webserver 的自动向注册中心注册
在服务的webserver中，开启一个自主向服务中心上报的服务。这里我们用redis来当做服务注册中心。用golang 来编写上报服务
```golang
package main

import (
	"fmt"
	"log"
	"net"
	"os"
	"strconv"
	"time"
	rds "webapp/module/redis"
)

var defaultSentinel rds.Sentinel = rds.DefaultSentinel

//注册服务的key
var ipFormat string = "/work/ips/%s"

func main() {
	//常驻后台
	for {
		//从系统变量中获取服务端口
		webappPort := os.Getenv("WEBAPP_PORT")
		if webappPort == "" {
			//默认服务端口为80
			webappPort = "80"
		}
		//如果设置的不是数字格式，程序退出
		_, err := strconv.Atoi(webappPort)
		if err != nil {
			log.Printf("webappPortErr:%v\n", err)
			// 防止运行过快cpu负载过高
			time.Sleep(time.Duration(5) * time.Second)
			continue
		}
		// 从系统变量中获取注册过期时间
		expireTime := os.Getenv("EXPIRE_TIME")

		expireTimeInt, err := strconv.Atoi(expireTime)
		if err != nil {
			//默认过期时间是1s
			expireTimeInt = 1
		}
		//注册服务
		err = registerServer(expireTimeInt)

		if err != nil {
			log.Printf("registerServerErr:%v\n", err)
			time.Sleep(time.Duration(5) * time.Second)
			continue
		}

		time.Sleep(time.Duration(5) * time.Second)
	}
}

//获取本机的ip地址
func getLocalIp() (string, error) {
	addrSlice, err := net.InterfaceAddrs()
	var IpAddr string
	if nil != err {
		return "", err
	}
	for _, addr := range addrSlice {
		if ipnet, ok := addr.(*net.IPNet); ok && !ipnet.IP.IsLoopback() {
			if nil != ipnet.IP.To4() {
				IpAddr = ipnet.IP.String()
				return IpAddr, nil
			}
		}
	}
	return "", fmt.Errorf("getLocalIpFaild")

}

//向存储中注册服务
// expireTime int 注册过期时间
func registerServer(expireTime int) error {
	var ipAddr string

	conn, err := defaultSentinel.GetRedisConn()
	if err != nil {
		return fmt.Errorf("get redis conn failed")
	}
	for i := 0; i < 2; i++ {
		ipAddr, err = getLocalIp()
		if err == nil {
			return fmt.Errorf("get localIp failed")
		}
	}
// 将ip地址加入redis中
	_, err = conn.Do("master", "SET", fmt.Sprintf(ipFormat, ipAddr), "EX", 5)
	if err != nil {
		return fmt.Errorf("requery redis failed")
	}
	return nil

}
```
值得注意的是：
- 常驻进程每5秒上报服务ip和服务端口，redis 设置的过期时间也是5秒，这样能保证服务能及时更新，具有更高的时效性,这个时间只是我自己随手写的，具体的事假可以自己进行实际情况进行设置
- 这里依旧采用了第二章的sentinel获取redis master的地址，保证了注册中心的高可用性

把上报服务添加进webapp的docker中，其实类比于上生产环境的上线
```shell

```
更改我们的启动脚本，现在既要启动webapp ，又要起服务上报进程
脚本内容如下
```shell
```

### 负载均衡组件的自动更新
在上面可知，负载均衡是由nginx所做，更加细致点来看的话，通过改变nginx的配置，能达到及时动态缩扩容的效果
confd是一个比较简单的配置管理工具，通过预设模板和添加命令，来达到动态更新配置的功能
为了方便部署，我们直接建造个 nginx + confd的镜像吧
下面是dockerfile
```shell
From nginx

RUN   mkdir -p /etc/confd/conf.d && mkdir -p /etc/confd/templates 

COPY ./conf/webapp_nginx.conf.toml /etc/confd/conf.d

COPY ./conf/8220.conf.tmpl  /etc/confd/templates 

COPY ./confd /bin

COPY ./boost.sh /etc

run   chmod +x /etc/boost.sh 

```
这个dockerfile做了两件事情
1. 将confd bin和配置文件打包进nginx的镜像中
2. 将启动脚本也打包进去
~~其实 docker-compose 能达到一样的目的，这里只是为了让大家多了解一个新的解决方案~~
执行 `docker build -t nginxconfd` 在用 `docker images ` 如图，我们新的nginxconfd 镜像生成了
{% img /images/3_docker_ret_1.png %}

下面 confd 的模板文件
```shell
```
这个是生成nginx的conf模板
```shell
upstream webapp{
{{range getvs "/work/webapp/ips/*"}}  # confd 会自动填充此部分参数
    server {{.}} weight=10;
{{end}}
}

server {
    listen       8220;
    server_name  webapp;

    #charset koi8-r;
    access_log  /var/log/nginx/host.access.log  main;

    location / {
        proxy_pass  http://webapp;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
 }

```
这个是confd的配置文件，设置了路径，回调命令。如果，在更新模板后，会reload nginx 配置，从而生效

```shell
[template]
src = "8220.conf.tmpl"
dest = "/etc/nginx/conf.d/8220.conf"
keys = [
  "/work/webapp/ips",
]
check_cmd = "/usr/sbin/nginx -t -c {{.src}}"
reload_cmd = "/usr/sbin/service nginx restart"
```
我们将nginxconfd替代之前的nginx容器重新部署
```shell

```
其启动脚本为
```shell
#! /bin/bash

 nginx -g "daemon off ;"  &
 confd  -interval 10 -backend redis -node redis_master:6379 
```
这里我们重点介绍下这条命令
```shell
confd  -interval 10 -backend redis -node redis_master:6379 
```
将confd 启动 并将 每个 6秒 （-interval） 从 redis_master:6379 (-node) 拉取上面tmplate中key的值，用于生产新的confd。并执行相应的回调命令
至此，一个服务中心已经搭建完成
#### 实验验证
我们将一个webapp_4加入集群中
```shell
webapp_4:
        image: golang
        container_name: webapp_4
        ports:
            - 8084:80
        networks:
            - front
        volumes:
            - ./go/webapp:/go/src/webapp
            - ./sh:/sh
        command: sh /sh/start.sh 
 
```
使用`  xx` 查看nginxconfd的配置
{% img images/3_docker_ret_2.png %}
配置已经将webapp_4的ip地址加进去

随机访问下，从nginx access_log中，证明了4台webapp都是在进行服务
{%img images/3_docker_ret_3.png %}

## 结论
至此，负载平衡和简单的注册中心,已经搭建完成，此时的架构为
{% img images/3_webapp_struct_2.png %}



