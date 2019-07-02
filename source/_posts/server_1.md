---
title: 服务的搭建及其演变 -简单的单点服务搭建
date: 2017-06-28 00:00:00
tags: webapp docker mysql golang
---

# 服务的搭建及其演变 -简单的单点服务搭建

在日常构建webapp应用中，最常见的就是一个代码模块，依赖一个数据库的架构。我们以一个简单的点赞系统来介绍架构是怎么一步一步由简单变得复杂的。 为了更好的在单机模拟分布式环境，也为了后续的方便部署，我引入的了docker。

{% img /images/1_alone.png%}
上图是最简单 webapp 依赖单db的架构图

  ## 本节相关工具
  - golang : 一种接近于c的编程语言。在开发难度和性能上做到了比较好的平衡。采用协程+管道的设计，天然较好地支撑了并发,是目前较火的编程语言,这里主要用于编写代码逻辑。
  - mysql  :  老牌的开源数据库,这里用于数据的持久化存储。
  - docker ： 近年较火的容器，主要综合和标准化了linux 下的namespace 和 cgroup 技术，并提供相应的手脚架和多种丰富的镜像用于环境的快速部署，资源隔离。本文主要用docker来进行环境的搭建和分布式的模拟
      下面简单了解下docker比较重要的概念
1. 镜像 : 打包好的软件合集，是创建容器的模板
2. 容器 ：主要运行时候的实体，和镜像类似于 类和对象的关系
3. 仓库 ：镜像市场，可以在上面下载到各种各样的镜像

 上面的解释纯基于我个人理解，如果有不到位的地方请谅解,相关工具的使用方法，我会在使用中说清楚。具体的深度使用细节，可以参考相关的专业书籍。

## 开始搭建
实验环境主要分为以下几步：
- docker,go 环境的安装，此类教程网上很多，这里就不在赘述
- 基于docker 的mysql 环境搭建
- app代码的编写
- 基于docker的app部署
### 基于docker的mysql 环境搭建
#### 数据库的搭建
``shell
ocker network create front
``
创建基于bridge模式名称为front的网络，后面我们会将所有部署的docker 容器添加进来

```shell
  docker run -d --name mysql -p 3306:3306  --env MYSQL_ROOT_PASSWORD=123456 --network front mysql
```
分析下这条docker命令的构成:

- docker run 是启动一个容器的命令
- -d 参数指明是后台运行此容器
- --name 是给容器命名，这里叫做 mysql
- -p 指明的是容器内端口和宿主机器端口的相互映映射,这里将宿主 3306  端口映射到容器的3306端口,一般mysql服务都使用3306端口
- --network 参数表明要添加到哪个网络中，这里加入上面创建的front网络中
- mysql 这个表明要在哪个镜像上创建容器，这里选择的是mysql的原生镜像。当镜像存在是会直接使用，不然docker eneeger 会替我们从网上进行下载

- --env 参数表面要设置的环境变量，采用的是key=value的参数形式，我们这里把mysql root 的密码设置为 123456
	  
输入 `docker containers ls -a `
	  
{% img /images/1_docker_show.png%}
	  
看到我们的mysql容器已经搭建完成
#### 数据库管理员及表结构的创建
```shell
 mysql  -h127.0.0.1 -P3306 -uroot -p123456
```
在我们的宿主机器上登录我们的mysql，如果你的宿主机器内没有mysql 客户端，也可以用如下命令登录 mysql 容器内，进行mysql的管理
```
   docker exec -it  mysql /bin/bash
```
执行下面sql语句
```sql
create database praise;
```
创建一个点赞数据库

```sql
CREATE USER 'test'@'%' IDENTIFIED BY 'test';
GRANT ALL ON *.* TO 'test'@'%';
```
创建一个用户名和密码都是 test 的账号,并给此账号赋予最高权限


```sql
ALTER USER 'test'@'%' IDENTIFIED WITH mysql_native_password BY 'test';
```
更改这个用户的验证方式，这一步的目的主要是解决某些低版本的mysql客户端验证不通过的问题。

```sql
CREATE TABLE `praise_count` (
 `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT 'id',
 `resource_id` bigint(20) NOT NULL DEFAULT '0' COMMENT '资源id',
 `count` bigint(20) NOT NULL DEFAULT '0' COMMENT '次数',
 `ctime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'create time',
 `mtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'modify time',
 PRIMARY KEY (`id`),
 UNIQUE KEY `resource_id` (`resource_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1968180 DEFAULT CHARSET=latin1 COLLATE=latin1_bin ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8 COMMENT='praise_count'

```
创建 点赞 数据表结构
&emsp;
**到这里，我们一个可用于项目的数据库已经搭建完成**
&emsp;
*本教程放在真正的生产环境中需要注意的是：*
1. docker并不适用存储类的服务，存储类服务一般都占用大量磁盘，难以迁移没有必要部署在容器中。这里只是方便搭建环境，所以采用。
2. 数据库的账号给予权限过高，密码过于简单，存在安全风险。


### app 代码的编写
这里的app相对比较简单，只实现两个接口。
```
` /praise/set?resource_id={int}`
  点赞次数自增
` /praise/get?resource_id={int}`
  查询点赞次数

resource_id get参数，传入点赞资源的唯一id
```
本项目除了依赖mysql的驱动，其余都使用go的原生工具
```shell
go get -u github.com/go-sql-driver/mysql
#安装go 的mysql驱动
```
下面进入代码
```go
package main

import (
        "database/sql"
        "encoding/json"
        "flag"
        "fmt"
        "io"
        "log"
        "net/http"
        "strconv"

        _ "github.com/go-sql-driver/mysql"
)

//c/s协议返回的json的结构体
// {
//    errno : -1/0, 错误码，0代表正常
//    errmsg: "" , 错误信息
//    data : {
//             resourceID : 111, 资源id，用于表示唯一的一条资源
//             count      : 1 ,  此资源被点赞的次数
//             Ctime      : 1231231, 此项被创建的时间
//  }
//

type Response struct {
        Errno  int    `json:"errno"`
        ErrMsg string `json:"errmsg"`
        Data   *Item  `json:"data"`
}
type Item struct {
        ResourceID int64 `json:"resourceID"`
        Count      int64 `json:"count"`
        Ctime      int64 `json:"ctime"`
}
// 数据库地址
var dbAddr string
// 数据库端口
var port int

func init() {
         // 主要是用于支持可自定义传入数据库的地址和端口号
		 // 例如 go run main.go -mysql=mysql -port=3304 将数据库地址和端口号传入。此处也写有默认值
        flag.StringVar(&dbAddr, "mysql", "mysql", "please input your mysql address,exp : 127.0.0.1")
        flag.IntVar(&port, "port", 3306, "please input your mysql port,exp : 3304")
}
//测试接口
func HelloServer(w http.ResponseWriter, req *http.Request) {
        io.WriteString(w, "hello, world!\n")
}

func main() {
        //命令端参数接解析
        flag.Parse()
        log.Println("server init")
		//注册三个接口，分别是 服务器测试接口，点赞接口，和点赞数获取接口
        http.HandleFunc("/hello", HelloServer)
        http.HandleFunc("/praise/get", getPrasieCount)
        http.HandleFunc("/praise/set", setPrasieCount)
        log.Println("server start")
		//启动服务器，并监听80端口
        log.Fatal(http.ListenAndServe(":80", nil))
}

// 获取点赞数，主要逻辑
func getPrasieCount(w http.ResponseWriter, req *http.Request) {
        resourceID, err := getResourceIDFromGet(req)
        if err != nil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", err))
                return
        }

        dbOpenCommand := fmt.Sprintf("test:test@tcp(%s:%d)/praise?charset=utf8", dbAddr, port)
        db, err := sql.Open("mysql", dbOpenCommand)

        if err != nil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", err))
                return
        }
        defer db.Close()

        sql := fmt.Sprintf("select count,ctime from praise_count where resource_id = %d", resourceID)
        rows, sqlerr := db.Query(sql)
        if sqlerr != nil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", sqlerr))
                return
        }

        defer rows.Close()
        res := Response{Errno: 0}
        var count, ctime int64
        if rows.Next() {
                rows.Scan(&count, &ctime)
        }
        res.Data = &Item{ResourceID: resourceID, Count: count, Ctime: ctime}

        retBytes, _ := json.Marshal(res)

        io.WriteString(w, string(retBytes))

}
//添加点赞数
func setPrasieCount(w http.ResponseWriter, req *http.Request) {
        resourceID, err := getResourceIDFromGet(req)
        if err != nil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", err))
                return
        }

        dbOpenCommand := fmt.Sprintf("test:test@tcp(%s:%d)/praise?charset=utf8", dbAddr, port)
        db, err := sql.Open("mysql", dbOpenCommand)
        if err != nil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", err))
                return
        }
        defer db.Close()
        sql := fmt.Sprintf("INSERT INTO praise_count  (resource_id,count ) VALUES (%d,1) ON DUPLICATE key UPDATE count=count+1", resourceID)
        _, err = db.Query(sql)
        if err != nil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", err))
                return
        }
        retBytes, _ := json.Marshal(Response{Errno: 0})
        io.WriteString(w, string(retBytes))

}

//错误处理
func returnErrMsg(w http.ResponseWriter, errno int, errmsg string) {

        retBytes, _ := json.Marshal(Response{Errno: errno, ErrMsg: errmsg})
        io.WriteString(w, string(retBytes))
}
//解析get参数，获取资源ID
func getResourceIDFromGet(req *http.Request) (int64, error) {
        vars := req.URL.Query()
        resourceIDStr := vars.Get("resource_id")
        resourceID, err := strconv.ParseInt(resourceIDStr, 10, 64)
        return resourceID, err
}
```

一个可以上线的代码已经编写完成。接下，将其打包到docker里。你可以想象，docker就是我们的一个服务器，或者是一个pass平台

``` shell
#启动docker的命令
docker run -d -p 8080:80 -v $HOME/cluster-build/chapter_1/src/go/webapp:/go/src/webapp -v $HOME/cluster-build/chapter_1/src/sh:/sh --network front golang sh /sh/start.sh
```
这个docker run 命令较为复杂,很多参数的含义我们在上面已经介绍过了，我们介绍没有见过的吧

- -v 文件挂载，将宿主机器上的文件挂载到容器中。其中我们把宿主机器的$HOME/cluster-build/chapter_1/src/go/webapp和 $HOME/cluster-build/chapter_1/src/sh文件夹分别映射到容器/go/src/webapp /sh下。多个 -v 参数可以进行多次映射。
- --network 参数表明要添加到哪个网络中，这里加入上面创建的front网络中，和mysql处于同一个网络中，方便通讯
- golang 这个表明要在哪个镜像上创建容器，这里选择的是golang的原生镜像。当镜像存在是会直接使用，不然docker eneeger 会替我们从网上进行下载
- sh start.sh 后置命令，当容器创建完成是，执行这个命令，这里执行一个手写的脚本

start.sh 这个后置脚本主要做两件事情：
1. 下载相关的依赖
2. 将服务启动
```shell
go get -u github.com/go-sql-driver/mysql  &&  go run /go/src/webapp/main.go
```
大家看到 上面的挂载命令很复杂，一般大家都会把自己基于golang镜像，再打包出一个新的镜像。这里我们为了直观和方便调试，先采用这种方式进行挂载。后面，我们会用docker-compose 来管理所有的docker容器。

# 结果验证
  
   用curl来访问我们期待的服务,资源id 为1 ，先访问 set 接口，在访问 set接口，结果为：
- set 接口请求示例

{% img /images/1_web_set.png%}
- get 接口请求示例
{% img /images/1_web_get.png%}

到这里，最简单的单点服务就已经完成.
