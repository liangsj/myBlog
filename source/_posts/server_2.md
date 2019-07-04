---
title: 服务的搭建及其演变 - 高可用分布式缓存系统构建
date: 2019-06-28 00:00:00
categories:
 - 编程
tags: 
    - webapp
    - docker-compose 
    - redis 
    - sentinel
---
# 前言
在前一节中，已经实现了最简单的单点模式webapp 搭建。这种架构往往很难抗住高qps（每秒访问次数）的冲击。在业务发展，往往要进行改造。一套抗高访问量的系统往往是很复杂的，每一个组件都有其优化的点。在众多的环境中，数据库往往是最薄弱的一环. 
对于数据库优化，目前有几种
- 数据库上流进行缓存。将热数据存入内存数据库中，如常见的nosql数据库 redis、memcache；或者添加到webapp 的localcache中。这种方式优点是: 部署简单；执行简单。缺点是 入侵代码，增加代码复杂度；缓存数据存在不一致的风险；缓存命中率如果无法保证的话，也达不到保证下游安全的目的。
- mysql 进行主从部署，一般来说主库只负责写操作，并将数据库内容同步更新到从库中。从库负责读操作。优点时：对代码入侵程度小；当某个库不可用时，我们可以进行切换。缺点是：在大规模的写操作时，可能会带来主从数据的延迟；主库压力大。
- 进行拆库拆表。 mysql 主从部署解决不了当数据急剧增加上，查询、插入过慢的问题。这时候一般我们会进行拆库拆表，这是一个痛苦的过程。其对业务代码入侵非常大。
上述几个方法，可能会同时存在。在这里介绍下，最常见的添加缓存的方案。并进行扩展，描述 redis 怎么进行主从部署以及自动进行主从切换（mysql 主从部署的大体也是这么一个流程，就不重复介绍了）。

# 相关工具
本节会用到新的三个伙伴
- redis ： 基于内存的 key-value 存储系统,能承受较高的qps，支持本地磁盘持久化备份。采用单线程的设计方案，逻辑相对简单。
- docker-compose : docker 相关的手脚架工具，可以定义 、部署多个docker 容器。
（相信大家到我前一节又长又臭的docker run 命令已经有些无奈了。docker - compose 能帮助我们解决这个问题）
- sentinel : 用于管理、监控 以及故障迁移 redis的工具

# 分布式环境构建
## 缓存设计 -  基于docker单独redis搭建
这里引入 docker-compose 来帮助我们管理日益复杂的docker 容器.docker-compose 安装 `sudo pip install -U docker-compose`
安装好后，创建一个名为 docker-compose.yml文件 `touch docker-compose.yml`,并往文件内添加以下内容
```yml
version: '2.0' # docker-compose.yml 的版本信息,这里写成2.X
services:   # 要定义的服务信息，这里除了需要添加之前的webapp 和 mysql 的运行环境，还需要添加redis服务
   webapp:
        image: golang       # 镜像名称
        container_name: webapp # 生成的容器名称
        ports:
            - 8080:80      # 宿主机器和容器的端口映射，这里是 宿主机器端口号:容器端口号
        networks:
            - front        # 此容器添加入的网络
        volumes:
            - ./go/webapp:/go/src/webapp   # 宿主机器目录和容器目录的映射， 这里是 宿主机器目录:容器内目录
            - ./sh:/sh
        command: sh /sh/start.sh       # 容器创建后要运行的命令
```
上面这段配置相当于 上节的 `docker run -d -p  8080:80 -v $HOME/src/go/webapp:/go/src/webapp -v $HOME/src/sh:/sh --network front golang sh /sh/start.sh`
```yml     
   mysql:
        image: mysql
        container_name: mysql
        ports:
            - 3306:3306
        networks:
            - front
        environment:
            "MYSQL_ROOT_PASSWORD": "123456" # 容器内要设置的系统变量，这里是设置容器的mysql root密码
   redis_master:
        image: redis
        container_name: redis_master
        networks:
            - front
        ports:
            - 7002:6379
        restart: always

  networks:     # 定义网络信息
    front: # 新网络名称
        driver: bridge # 网络的模式,一般都选择bridge （docker中有多种网路模式，可以根据不同的使用场景进行使用，想深入了解，可以看docker官网介绍）
```
webapp 中需要操作redis，这里依赖redigo这个开源的库，在启动之前把依赖下载的指令加入之前的start.sh 中,start.sh现在为

```shell
go get -u github.com/go-sql-driver/mysql && go get -u github.com/garyburd/redigo/redis  &&  go run /go/src/webapp/main.go
```
docker-compose.yml 完成后，在当前目录下执行 `docker-compose up -d`,常见容器<br>
输入 `docker container  ls -a`
{%img /images/2_compose_ret.png%}
看到我们的redis，mysql，webapp已经跑起来了

## 代码改写
基本的开发环境已经搭建完毕，现在改写我们之前的webapp 代码<br>
这里先添加 三个关于缓存操作的代码
- 添加缓存
```go
//将点赞数放入redis_master缓存中
func setToCache(resourceID int64, praiseCount int64) error {
        conn, err := redis.Dial("tcp", "redis_master:6379") //连接redis, host:port形式，host是上面定义的容器名称，port 是redis容器的redis服务端口号
        if err != nil {
                return err
        }
        defer conn.Close()
        //const praise_count_cache_key_fmt string = "resource_%d_prasie_cache"
        key := fmt.Sprintf(praise_count_cache_key_fmt, resourceID) 
        _, err = conn.Do("SET", key, praiseCount) //将结果放入redis中
        return err

}
```
- 清除缓存
```
//清除缓存数据
func CleanCache(resourceID int64) error {
        conn, err := redis.Dial("tcp", "redis_master:6379")
        if err != nil {
                return err
        }
        defer conn.Close()
        key := fmt.Sprintf(praise_count_cache_key_fmt, resourceID)
        _, err = conn.Do("DEL", key)
        return err
}
```go
- 从缓存中获取数据
``` go
//从redis_master中获取点赞数
func getFromCache(resourceID int64) (int64, error) {
        conn, err := redis.Dial("tcp", "redis_master:6379")
        if err != nil {
                return 0, err
        }
        defer conn.Close()
        key := fmt.Sprintf(praise_count_cache_key_fmt, resourceID)
        praiseCount, err := redis.Int64(conn.Do("GET", key))
        if err != nil {
                return 0, err
        }
        return praiseCount, nil
}
```
接下来修改主流程里面的代码
删除缓存操作放在写操作内，保证缓存一致
```go
func setPrasieCount(w http.ResponseWriter, req *http.Request) { 
       ..... //省略代码，如需要看上一节，在git中下载
        _, err = db.Query(sql)
        if err != nil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", err))
                return
        }
        CleanCache(resourceID) //在添加点赞数是删除缓存
        retBytes, _ := json.Marshal(Response{Errno: 0})
        io.WriteString(w, string(retBytes))

}
```
读取缓存
```go
func getPrasieCount(w http.ResponseWriter, req *http.Request) {
        ... //省略参数获取部分
        praiseCount, err := getFromCache(resourceID)  // 从缓存读取点赞数
        if err != nil && err != redis.ErrNil {
                returnErrMsg(w, -1, fmt.Sprintf("%v", err))
                return
        }

        if err != redis.ErrNil { //当缓存内有数据时直接返回

                res := &Response{Errno: 0, Data: &Item{ResourceID: resourceID, Count: praiseCount}}
                retBytes, _ := json.Marshal(res)
                setToCache(resourceID, praiseCount)
                io.WriteString(w, string(retBytes))
                return

        }
        ......//省略从数据库读取数据部分
        
        setToCache(resourceID, count) //缓存内没有数据，将从数据库里面读到的数据放入缓存中，以便下次使用
        retBytes, _ := json.Marshal(res)
        io.WriteString(w, string(retBytes))
}
```
从上述代码中，可以看到，在读取点赞数时，先读缓存，如果缓存有数据我们直接当做结果进行返回。如果结果中没有数据，在进行数据库查询，并将查询结果放回redis-cache中。当有足够的高的缓存命中率时，能很好减少到下游db的流量，从而达到保护db的目的<br>
在写点赞数时，会把redis cache的数据进行删除，从而保证下次直读db，保证数据的一致性。***存在缓存设计的风险点，当缓存删除失败时，会造成缓存数据和db数据不一致，对于要求数据强一致的业务不能这么进行设计***
## 主从结构的分流设计 - 基于docker带有主从结构的redis 搭建
至此，保护db的目的已经达到了。假设缓存流量持续上涨，缓存命中率也较高的情况下。redis-cache会成为新的瓶颈，除了在redis -cache 上在加一层在服务器上的local-cache外。我们还有第二个解决方案 ，进行主从结构的部署,（这里提供redis的解决方案，db也可以参考这种方案）。 redis 本身就支持主从结构的部署，只需要简单的命令 `redis-server --slaveof <$redis_master_host> <$redis_master_port>` 即可<br>

下面解决往docker-compose.yml添加新服务
```yml
 redis_slave_1:
        image: redis
        ports:
            - 7003:6379
        restart: always
        container_name: redis_slave_1
        command: redis-server --slaveof redis_master 6379 #启动一个redis服务，并设置成为redis_master的从库
        networks:
            - front
        image: redis
        restart: always
   redis_slave_2:
        image: redis
        ports:
            - 7004:6379
        restart: always
        container_name: redis_slave_2
        networks:
            - front
        image: redis
        restart: always
        command: redis-server --slaveof redis_master 6379
```
修改完成后，执行 `docker-compose up -d` 镜像重新部署
{%img /images/2_compose_ret2.png%}
结果显示，我们两个redis 的从库已经部署好了
### 代码改写
下面进行读写入口的改写
```go
func getFromCache(resourceID int64) (int64, error) {
        conn, err := redis.Dial("tcp", "redis_slave_1:6379") //更改读逻辑的redis入口
        if err != nil {
                return 0, err
        }
        ....//省略其他逻辑
       return praiseCount, nil
}
```
最后，写缓存流量在redis_master 主库上，读流量在从库上（从库如果有写操作会进行保存），大大减少redis master主库上的流量，从而达到分流的目的

## 自动主从切换 - 基于docker的sentine 环境搭建
通过以上两个设计，系统稳定性已经上升了一个档次。但，我们观察到，主库现在又面临着单点的问题。如果主库出现可用性问题，结果往往是灾难的。我们需要套机制来进行监控主库和稳定性和当出现问题时，能进行主从切换,我们依旧以redis为例
sentinel作为最常用的redis 监控、主从切换工具而被广泛应用。首先，先进行sentinel配置的编写,保存为 sentinel.conf
```shell
port 26379 # sentinel 的服务端口

dir /tmp  # 工作文件目录

sentinel monitor master redis_master 6379 1 # 对redis主库进行监听 后面三个参数的意思分别是 ： 主库的host，主库的端口，当 1 个sentinel 检查出现错误后，自动进行主从切换
sentinel down-after-milliseconds master 30000 # 心跳检查，当主库在30000 ms内没有应答，则认为其已经不可用，进行容灾操作
```

### 代码改写
### 结果验证

## 生产环境中的注意点






