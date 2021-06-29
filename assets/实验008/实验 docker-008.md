# Overlay网络实验

基本要求：至少拥有一个单节点的docker swarm集群。

本次实验推荐配置：

2核4G CentOS 7 * 3

- docker01
- docker02
- docker03

## 1. 构建swarm集群

所有三个docker daemon都将加入到集群中，集群分为两个主要角色`master`和`worker`。

我们将以docker01作为master（manager）

在docker01上初始化swarm

```bash
[root@docker01 ~]#  docker swarm init --advertise-addr=192.168.238.134
Swarm initialized: current node (8fztp5t95jiaxtmrxfc70fi6y) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-67he4scmfkbe9jprg947q1fmtnnsbldm6a2r67v5xelezdpujm-2ivh20o4fx5qwgyih8bseuxmf 192.168.238.134:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

将docker01和docker02加入到集群中

docker01

```bash
[root@docker02 ~]# docker swarm join --token SWMTKN-1-67he4scmfkbe9jprg947q1fmtnnsbldm6a2r67v5xelezdpujm-2ivh20o4fx5qwgyih8bseuxmf 192.168.238.134:2377
This node joined a swarm as a worker.
```

docker02

```bash
[root@docker03 ~]# docker swarm join --token SWMTKN-1-67he4scmfkbe9jprg947q1fmtnnsbldm6a2r67v5xelezdpujm-2ivh20o4fx5qwgyih8bseuxmf 192.168.238.134:2377
This node joined a swarm as a worker.
```

查看集群状态

```bash
[root@docker01 ~]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
8fztp5t95jiaxtmrxfc70fi6y *   docker01            Ready               Active              Leader              19.03.13
0wxad6m9fh0osbiylowawak8e     docker02            Ready               Active                                  19.03.13
ici4a0h3gje03bjnnk1l978vm     docker03            Ready               Active                                  19.03.13
```

查看容器网络

```bash
[root@docker01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
2025ab2f9945        bridge              bridge              local
d447cf391c1b        docker_gwbridge     bridge              local
71d591cd4094        host                host                local
njl7w97vfqyi        ingress             overlay             swarm
a594d649a531        none                null                local
```

`docker_gwbridge`链接到`ingress`网络，关联到host网络接口，以实现流量在manager和worker之间传输。创建swarm service，如果不指定网络，则会自动连接到ingress中。推荐为不同service创建不同的overlay网络或将相关的service接入指定的ingress中。

## 2. 创建service

首先创建自己的overlay网络

```bash
[root@docker01 ~]# docker network create -d overlay nginx-net
qyra1ofn0y9hcobge0o9c44vf
[root@docker01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
2025ab2f9945        bridge              bridge              local
d447cf391c1b        docker_gwbridge     bridge              local
71d591cd4094        host                host                local
njl7w97vfqyi        ingress             overlay             swarm
qyra1ofn0y9h        nginx-net           overlay             swarm
a594d649a531        none                null                local
```
不必在其他节点上创建该网络，当启动service stack时会在特定节点上自动创建网络

创建5副本的nginx服务

```bash
docker service create --name nginx --publish target=80,published=80 --replicas=5 --network nginx-net nginx
```

查看service，查看详情
```bash
[root@docker01 ~]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
o1m1mbpgxpcm        nginx               replicated          5/5                 nginx:latest        *:80->80/tcp

[root@docker01 ~]# 
[
    {
        "ID": "o1m1mbpgxpcm5ul7uunsagwen",
        "Version": {
            "Index": 24
        },
        "CreatedAt": "2020-11-25T11:36:13.529821588Z",
        "UpdatedAt": "2020-11-25T11:36:13.530421954Z",
        "Spec": {
            "Name": "nginx",
            "Labels": {},
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "nginx:latest@sha256:d2f64ceb0ed5a11537c0f6e151eafef2d1f95e910e0520aaa3eee4c3321a71ac",
                    "Init": false,
                    "StopGracePeriod": 10000000000,
                    "DNSConfig": {},
                    "Isolation": "default"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "Delay": 5000000000,
                    "MaxAttempts": 0
                },
                "Placement": {
                    "Platforms": [
                        {
                            "Architecture": "amd64",
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "Architecture": "arm64",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "386",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "mips64le",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "ppc64le",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "s390x",
                            "OS": "linux"
                        }
                    ]
                },
                "Networks": [
                    {
                        "Target": "qyra1ofn0y9hcobge0o9c44vf"
                    }
                ],
                "ForceUpdate": 0,
                "Runtime": "container"
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 5
                }
            },
            "UpdateConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "RollbackConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "EndpointSpec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 80,
                        "PublishMode": "ingress"
                    }
                ]
            }
        },
        "Endpoint": {
            "Spec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 80,
                        "PublishMode": "ingress"
                    }
                ]
            },
            "Ports": [
                {
                    "Protocol": "tcp",
                    "TargetPort": 80,
                    "PublishedPort": 80,
                    "PublishMode": "ingress"
                }
            ],
            "VirtualIPs": [
                {
                    "NetworkID": "njl7w97vfqyi3f3w900y73j9d",
                    "Addr": "10.0.0.5/24"
                },
                {
                    "NetworkID": "qyra1ofn0y9hcobge0o9c44vf",
                    "Addr": "10.0.1.2/24"
                }
            ]
        }
    }
]

```

创建一个新的网络nginx-net-2，将service迁移至新的网络
```bash
[root@docker01 ~]# docker service update --network-add nginx-net-2 --network-rm nginx-net nginx
nginx
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
```

尝试创建一个独立的容器使用overlay网络
```bash
[root@docker01 ~]# docker run -itd --name alpine1 --network nginx-net-2 alpine ash
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
188c0c94c7c5: Pull complete 
Digest: sha256:c0e9560cda118f9ec63ddefb4a173a2b2a0347082d7dff7dc14272e7841a5b5a
Status: Downloaded newer image for alpine:latest
docker: Error response from daemon: Could not attach to network nginx-net-2: rpc error: code = PermissionDenied desc = network nginx-net-2 not manually attachable.
ERRO[0024] error waiting for container: context canceled 
```

根据错误提示，我们不具备手动添加的能力

```bash
[root@docker01 ~]# docker service update --network-add nginx-net --network-rm nginx-net-2 nginx
nginx
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
[root@docker01 ~]# docker network rm nginx-net-2
nginx-net-2  
[root@docker01 ~]# docker network create nginx-net-2 --attachable -d overlay
mm0iumnqua25e7z7x3ls7ps60
```
重新迁回服务，重新尝试创建连接至nginx-net-2的独立alpine容器，测试服务连通性

```bash
[root@docker01 ~]# docker exec alpine1 ping -c 2 nginx
PING nginx (10.0.4.2): 56 data bytes
64 bytes from 10.0.4.2: seq=0 ttl=64 time=0.071 ms
64 bytes from 10.0.4.2: seq=1 ttl=64 time=0.077 ms

--- nginx ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.071/0.074/0.077 ms
```