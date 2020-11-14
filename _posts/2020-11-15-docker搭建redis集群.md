---
layout: post
title: docker搭建redis集群
date: 2020-11-15
tags: tools
---



### docker搭建redis集群

---



> docker下载地址：[docker官网](https://www.docker.com/get-started)

*请预先搭建好docker环境，使用docker命令拉取redis镜像文件。*这次一次搭建redis集群尝试多次失败后才成功步骤总结。

1. 构建目录redis/redis-cluster目录层次结构，在redis-cluster目录下添加redis模版文件 redis-cluster.tmpl，内容如下

```txt
port ${port}
## 不作为守护进程
daemonize no
## 启用aof持久化模式
appendonly yes

# 集群配置
## 开启集群配置
cluster-enabled yes
## 存放集群节点的配置文件 系统自动建立
cluster-config-file nodes-${port}.conf
## 节点连接超时时间
cluster-node-timeout 50000
## 实际为各节点网卡分配ip，根据自己服务器的主机定义
cluster-announce-ip 192.168.244.143 
## 节点映射端口
cluster-announce-port ${port}
## 节点总线端口
cluster-announce-bus-port 1${port}
cluster-slave-validity-factor 10
cluster-migration-barrier 1
cluster-require-full-coverage yes
```

**注意**：**cluster-announce-ip**:这个IP需要特别注意一下，如果要对外提供访问功能，需要填写宿主机的IP，如果填写docker分配的IP（172.x.x.x），可能会导致部分集群节点在跳转时失败。异常如下(大坑）：

```txt
JedisException: Could not get a resource from the pool
```

2. 创建虚拟网卡

```bash
docker network create redis-cluster-net
// 查看网卡详细信息
docker network inspect redis-cluster-net
```

3. 使用脚本create构建redis集群目录 

```bash
#!/bin/bash
com=$1
dir_redis_cluster='/home/admin/redis/redis-cluster'
# redis集群网关
gateway='172.18.0.1'

case ${com} in
	create)
        idx=1;
		for port in `seq 7000 7005`; do
            # 创建存放redis数据路径
			mkdir -p ${dir_redis_cluster}/${port}/data;
            # 通过模板个性化各个节点的配置文件
            idx=$(($idx+1));
            port=${port} ip=`echo ${gateway} | sed "s/1$/$idx/g"` \
                envsubst < ${dir_redis_cluster}/redis-cluster.tmpl \
                > ${dir_redis_cluster}/${port}/redis-${port}.conf
		done
	;;
    build)
        # 创建容器配置并运行
        for port in `seq 7000 7005`; do
            docker run --name redis-${port} --net redis-cluster-net -d \
            	-p ${port}:${port} -p 1${port}:1${port} \
                -v ${dir_redis_cluster}/${port}/data:/data \
                -v ${dir_redis_cluster}/${port}/redis-${port}.conf:/usr/local/etc/redis/redis.conf redis \
                redis-server /usr/local/etc/redis/redis.conf
        done
    ;;
    start | begin)
        # 运行容器
    	for port in `seq 7000 7005`; do
            docker start redis-${port}
        done
    ;;
    stop | end)
        # 停止容器运行
        for port in `seq 7000 7005`; do
            docker stop redis-${port}
        done
    ;;
    rm)
        # 删除已有容器
        for port in `seq 7000 7005`; do
            docker rm redis-${port}
        done
    ;;
    restart)
        # 重启已有容器
    	for port in `seq 7000 7005`; do
            docker restart redis-${port}
        done
    ;;
    destroy)
        # 删除集群目录及配置
        for port in `seq 7000 7005`; do
            rm -rf ${dir_redis_cluster}/${port}
        done
    ;;
    *)
        echo "Usage:	./build [create|build|start|stop|rm|restart|destroy]"
    ;;
esac
```

4. 使用脚本的build命令创建并执行容器
5. 查看redis集群状态，都为运行状态则为正常

```bash
docker ps

3036ea20c34a        redis               "docker-entrypoint..."   32 minutes ago      Up 32 minutes       0.0.0.0:7005->7005/tcp, 6379/tcp, 0.0.0.0:17005->17005/tcp   redis-7005
58e791b11181        redis               "docker-entrypoint..."   32 minutes ago      Up 32 minutes       0.0.0.0:7004->7004/tcp, 6379/tcp, 0.0.0.0:17004->17004/tcp   redis-7004
95c032c509b6        redis               "docker-entrypoint..."   32 minutes ago      Up 32 minutes       0.0.0.0:7003->7003/tcp, 6379/tcp, 0.0.0.0:17003->17003/tcp   redis-7003
079888b876a4        redis               "docker-entrypoint..."   32 minutes ago      Up 32 minutes       0.0.0.0:7002->7002/tcp, 6379/tcp, 0.0.0.0:17002->17002/tcp   redis-7002
1e5b7f1e1ddf        redis               "docker-entrypoint..."   32 minutes ago      Up 32 minutes       0.0.0.0:7001->7001/tcp, 6379/tcp, 0.0.0.0:17001->17001/tcp   redis-7001
88d93feaddf1        redis               "docker-entrypoint..."   32 minutes ago      Up 32 minutes       0.0.0.0:7000->7000/tcp, 6379/tcp, 0.0.0.0:17000->17000/tcp   redis-7000
```

6. 进入任意一个redis容器的bash终端，并构建redis集群

```bash
docker exec -it redis-7000 bash
// 进入后构建集群，会自动分配主副本和节点数
redis-cli --cluster create 192.168.244.143:7000 192.168.244.143:7001 192.168.244.143:7002 192.168.244.143:7003 192.168.244.143:7004 192.168.244.143:7005 --cluster-replicas 1

```

7. 查看redis集群状态

```bash
// 查看集群信息
docker exec -it redis-7000 redis-cli -p 7000 cluster info
// 查看集群节点信息
docker exec -it redis-7000 redis-cli -p 7000 cluster nodes
// 查看集群槽位分配信息
docker exec -it redis-7000 redis-cli -p 7000 cluster slots
```

8. 测试redis集群

```bash
docker exec -it redis-7000 redis-cli -h ip地址 -p 7000 -c 
// 输入命令检查redis节点跳转是否正常
set a b
get a
set c d
get c 
```

9. 编写java代码连接redis集群

```java
    public static void main(String[] args) {
        String addr = "192.168.244.143";
        Set<HostAndPort> nodes = new HashSet<>();
        nodes.add(new HostAndPort(addr, 7000));
        nodes.add(new HostAndPort(addr, 7001));
        nodes.add(new HostAndPort(addr, 7002));
        nodes.add(new HostAndPort(addr, 7003));
        nodes.add(new HostAndPort(addr, 7004));
        nodes.add(new HostAndPort(addr, 7005));
        GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
        poolConfig.setMaxTotal(8);
        poolConfig.setMaxIdle(8);
        poolConfig.setMinIdle(1);
        poolConfig.setMaxWaitMillis(1000);

        JedisCluster cluster =  new JedisCluster(nodes, 5000, 2000, 3, poolConfig);
      	System.out.println(cluster.get("yes"));
        System.out.println(cluster.get("hello"));
        cluster.set("test2", "6739");
        System.out.println(cluster.get("test2"));
    }
```





