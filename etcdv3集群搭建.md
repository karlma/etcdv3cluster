etcd v3 集群搭建
========

[TOC]


## Build docker image

```
git clone https://github.com/coreos/etcd.git
git checkout v3.2.6
docker build -t karl/etcd:v3.2.6 .
```

## 启动单机etcd docker

etcd在容器启动前就需要知道容器的ip地址，所以需要docker run时指定容器的ip地址。

指定docker容器的ip地址需要创建自己的network，不能用docker默认的network

1. 列表docker的network

```
$ docker network list

NETWORK ID          NAME                DRIVER              SCOPE
552708c14108        bridge              bridge              local
52631ae5ec73        host                host                local
99f7ffe66c0f        none                null                local
```

2. 创建自定义的network

```
$ docker network create --subnet=172.18.0.0/24 mynet
$ docker network list
NETWORK ID          NAME                DRIVER              SCOPE
552708c14108        bridge              bridge              local
52631ae5ec73        host                host                local
dc5ad6b73214        mynet               bridge              local
99f7ffe66c0f        none                null                local
```

3. 启动etcd

```
$ export HOST=192.168.1.64 
$ export NODE1=172.18.0.3
$ docker run --net mynet --ip ${NODE1} -p 2379:2379 -p 2380:2380 --name etcd karl/etcd:v3.2.6 --name node1 --initial-advertise-peer-urls http://${HOST}:2380 --listen-peer-urls http://${NODE1}:2380 --advertise-client-urls http://${HOST}:2379 --listen-client-urls http://${NODE1}:2379 --initial-cluster node1=http://${HOST}:2380 
```

4. 测试etcd

```
$ etcdctl cluster-health
$ etcdctl set /message hello
$ etcdctl get /message hello
```

## 启动etcd集群

* 脚本

```
#!/bin/bash

NODE_IP=(172.18.0.3 172.18.0.4 172.18.0.5)
PORT_2379=(2379 2381 2383)
PORT_2380=(2380 2382 2384)
NAME=(etcd1 etcd2 etcd3)
DATA_DIR=(/Users/karl/workspace/etcd_cluster/data1 /Users/karl/workspace/etcd_cluster/data2 /Users/karl/workspace/etcd_cluster/data3)

HOST=192.168.1.64

CLUSTER=${NAME[0]}=http://${HOST}:${PORT_2380[0]},${NAME[1]}=http://${HOST}:${PORT_2380[1]},${NAME[2]}=http://${HOST}:${PORT_2380[2]}

TOKEN=udesk123
CLUSTER_STATE=new

start_etcd() {

	docker run --net mynet --ip $1 \
		-p $2:2379 -p $3:2380 \
		--volume=$5:/etcd-data \
		--name $4 \
		-d \
		karl/etcd:v3.2.6 --data-dir=/etcd-data --name $4 \
		--initial-advertise-peer-urls http://${HOST}:$3 \
		--listen-peer-urls http://$1:2380 \
		--advertise-client-urls http://${HOST}:$2 \
		--listen-client-urls http://$1:2379 \
		--initial-cluster ${CLUSTER} \
		--initial-cluster-state ${CLUSTER_STATE} \
		--initial-cluster-token ${TOKEN}

}

start() {
	c=${#NODE_IP[*]}
	i=0
	while [ "$i" -lt "$c" ]
	do
		echo -n "${NAME[$i]} -- "
		start_etcd ${NODE_IP[$i]} ${PORT_2379[$i]} ${PORT_2380[$i]} ${NAME[$i]} ${DATA_DIR[$i]}
		((i++))
	done
}

stop() {
	for i in ${NAME[*]}
	do
		docker stop $i
	done
}

remove() {
	for i in ${NAME[*]}
	do
		docker rm $i
	done
}

case $1 in
	start )
		start
		;;
	stop )
		stop
		;;
	remove )
		remove
		;;
	* )
		echo "Usage: $0 start|stop|remove"
esac
```

## gRPC Proxy

```
./etcd grpc-proxy start --listen-addr 127.0.0.1:2379 --endpoints 192.168.1.64:2380,192.168.1.64:2382,192.168.1.64:2384
```

这种方式是v3推荐的，可以减轻服务器压力。

## Gateway

```
./etcd gateway start --listen-addr 127.0.0.1:2379 --endpoints 192.168.1.64:2379,192.168.1.64:2381,192.168.1.64:2383
```
Gateway方式只是简单的代理，不缓存，会增加延时。

## 本地启动集群

```
$ go get github.com/mattn/goreman
$ goreman -f Procfile start
```
Profile文件在源码目录下。