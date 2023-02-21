[etcd](https://github.com/coreos/etcd) 是 CoreOS 团队发起的一个开源项目（Go 语言，其实很多这类项目都是 Go 语言实现的，只能说很强大），实现了**分布式键值存储**和**服务发现**，etcd 和 ZooKeeper/Consul 非常相似，都提供了类似的功能，以及 REST API 的访问操作，具有以下特点：

- 简单：安装和使用简单，提供了 REST API 进行操作交互
- 安全：支持 HTTPS SSL 证书
- 快速：支持并发 10 k/s 的读写操作
- 可靠：采用 raft 算法，实现分布式系统数据的可用性和一致性

　　使用 docker 单机安装：

# 1、使用镜像 https://hub.docker.com/r/bitnami/etcd

```
docker pull bitnami/etcd:latest
```

　　　也可自己选择构建镜像

```
 docker build -t bitnami/etcd:latest 'https://github.com/bitnami/bitnami-docker-etcd.git#master:3/debian-10'
```

# 2、创建docker 网络

```
docker network create app-tier --driver bridge
```

# 3、启动etcd服务器实例

　　　　–network app-tier 指定启动实例的网络配置

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
docker run -d --name etcd-server \
    --network app-tier \
    --publish 2379:2379 \
    --publish 2380:2380 \
    --env ALLOW_NONE_AUTHENTICATION=yes \
    --env ETCD_ADVERTISE_CLIENT_URLS=http://etcd-server:2379 \
    bitnami/etcd:latest
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

# 4、启动etcd客户端实例，并连接上步骤服务

```
docker run -it --rm \
    --network app-tier \
    --env ALLOW_NONE_AUTHENTICATION=yes \
    bitnami/etcd:latest etcdctl --endpoints http://etcd-server:2379 set /message Hello
```

　　执行以上步骤时可能会遇到以下问题：**Error: unknown command "set" for "etcdctl"**

![img](https://img2020.cnblogs.com/blog/1110857/202110/1110857-20211024234456885-1407663610.png)

 　解决办法：

1.  默认v2版接口关闭，只需要把`set`改成`put`即可。

2. etcd开启v2版接口：在etcd的启动参数后面加上--enable-v2启动，或者在etcd的配置文件上加入ETCD_ENABLE_V2=true，然后重启etcd。使用etcdctl命令的时候前面加上ETCDCTL_API=2 ，调用v2接口。比如：ETCDCTL_API=2 etcdctl

   ```
   docker run -it --name etcd-server -p 2379:2379 -p 2380:2380 --env ALLOW_NONE_AUTHENTICATION=yes --env ETCD_ENABLE_V2=true -d bitnami/etcd
   ```

   

# 5、etcd 使用curl 实现键值管理操作的命令：**V2 版本**

5.1 设置键值命令：

```
curl http://127.0.0.1:2379/v2/keys/hello -XPUT -d value="hello world"
{"action":"set","node":{"key":"/hello","value":"hello world","modifiedIndex":17,"createdIndex":17}}
```

5.2 查看键值命令：

```
curl http://127.0.0.1:2379/v2/keys/hello
{"action":"get","node":{"key":"/hello","value":"hello world","modifiedIndex":17,"createdIndex":17}}
```

5.3 删除键值命令：

```
curl http://127.0.0.1:2379/v2/keys/hello -XDELETE

{"action":"delete","node":{"key":"/hello","modifiedIndex":19,"createdIndex":17},"prevNode":{"key":"/hello","value":"hello world","modifiedIndex":17,"createdIndex":17}}
```

5.4 查看版本：

```
curl -L http://127.0.0.1:2379/version

{"etcdserver":"3.5.1","etcdcluster":"3.5.0"}
```

5.4 判断leader和followers

```
curl http://127.0.0.1:2379/v2/stats/leader

{"leader":"8e9e05c52164694d","followers":{}}
```

