# 桥接网络实验

## 1.1 使用默认的桥接网络

查看当前容器网络
```bash
[root@docker ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
683f1b519a37        bridge              bridge              local
14d49050cbf4        host                host                local
9b4877e78f90        none                null                local
```

启动两个测试容器

```bash
[root@docker ~]# docker run -dit --name alpine1 alpine ash
[root@docker ~]# docker run -dit --name alpine2 alpine ash
```
检查容器是否正常启动

```bash
[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                            NAMES
26c47bfc6fb6        alpine              "ash"                    2 minutes ago       Up 2 minutes                                         alpine2
6f8cf6680d98        alpine              "ash"                    2 minutes ago       Up 2 minutes                                         alpine1
```

查看当前网卡状态

```bash
[root@docker ~]# brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024247fa8dcc       no              veth239a1c0
                                                        vethf62b5f2
                                                        vethf988cbf


[root@docker ~]# ip a
...
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:47:fa:8d:cc brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:47ff:fefa:8dcc/64 scope link 
       valid_lft forever preferred_lft forever
103: vethf988cbf@if102: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether ca:e5:e1:37:c7:31 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::c8e5:e1ff:fe37:c731/64 scope link 
       valid_lft forever preferred_lft forever
165: veth239a1c0@if164: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 4a:ae:fb:50:2c:cf brd ff:ff:ff:ff:ff:ff link-netnsid 4
    inet6 fe80::48ae:fbff:fe50:2ccf/64 scope link 
       valid_lft forever preferred_lft forever
167: vethf62b5f2@if166: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 1a:82:49:0a:ce:3e brd ff:ff:ff:ff:ff:ff link-netnsid 5
    inet6 fe80::1882:49ff:fe0a:ce3e/64 scope link 
       valid_lft forever preferred_lft forever


[root@docker ~]# docker exec alpine1 ip a show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
164: eth0@if165: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:06 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.6/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
[root@docker ~]# docker exec alpine2 ip a show 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
166: eth0@if167: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:07 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.7/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

```

测试第二个容器于第一个容器的连通性

```bash
[root@docker ~]# docker exec alpine2 ping 172.17.0.6 -c 2
PING 172.17.0.6 (172.17.0.6): 56 data bytes
64 bytes from 172.17.0.6: seq=0 ttl=64 time=0.077 ms
64 bytes from 172.17.0.6: seq=1 ttl=64 time=0.110 ms

--- 172.17.0.6 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.077/0.093/0.110 ms

```

使用第一个容器的name，在第二个容器内测试连通性

```bash
[root@docker ~]# docker exec alpine2 ping alpine1 -c 2          
ping: bad address 'alpine1'
```

***不利于编排，不建议用于生产环境***


## 1.2 使用用户自定义网桥

创建alpine-net，自定义网桥

```bash
[root@docker ~]# docker network create --driver bridge alpine-net
ffdf806b98d52c202e66602b3866cd316f63eceb7db2ac02f0d056c99b29d741
```
列出docker的网络

```bash
[root@docker ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
ffdf806b98d5        alpine-net          bridge              local
683f1b519a37        bridge              bridge              local
14d49050cbf4        host                host                local
9b4877e78f90        none                null                local
```

查看自定义桥详情

```bash
[root@docker ~]# docker network inspect alpine-net
[
    {
        "Name": "alpine-net",
        "Id": "ffdf806b98d52c202e66602b3866cd316f63eceb7db2ac02f0d056c99b29d741",
        "Created": "2020-11-19T10:13:47.129887971+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

创建4个容器

```bash
[root@docker ~]# docker run -dit --name alpine1 --network alpine-net alpine ash
[root@docker ~]# docker run -dit --name alpine2 --network alpine-net alpine ash
[root@docker ~]# docker run -dit --name alpine3 alpine ash
[root@docker ~]# docker run -dit --name alpine4 --network alpine-net alpine ash
[root@docker ~]# docker network connect bridge alpine4
```

再次查看自定义桥详情，自定义的alpine-net，不仅可以通过容器ip互访，还可以通过容器名称对附加在此网桥上的设备访问。

执行如下测试

```bash
[root@docker ~]# docker exec alpine1 ping -c 2 alpine2
PING alpine2 (172.19.0.3): 56 data bytes
64 bytes from 172.19.0.3: seq=0 ttl=64 time=0.095 ms
64 bytes from 172.19.0.3: seq=1 ttl=64 time=0.079 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.079/0.087/0.095 ms


[root@docker ~]# docker exec alpine1 ping -c 2 alpine4
PING alpine4 (172.19.0.4): 56 data bytes
64 bytes from 172.19.0.4: seq=0 ttl=64 time=0.065 ms
64 bytes from 172.19.0.4: seq=1 ttl=64 time=0.204 ms

--- alpine4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.065/0.134/0.204 ms


[root@docker ~]# docker exec alpine1 ping -c 2 alpine3
ping: bad address 'alpine3'


[root@docker ~]# docker exec alpine1 ping -c 2 172.17.0.3
PING 172.17.0.3 (172.17.0.3): 56 data bytes

--- 172.17.0.3 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss

```

从alpine4尝试与其他容器的联通

```bash
[root@docker ~]# docker exec alpine4 ping -c 2 172.17.0.3 
PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.108 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.086 ms

--- 172.17.0.3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.086/0.097/0.108 ms
[root@docker ~]# docker exec alpine4 ping -c 2 alpine1
PING alpine1 (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.046 ms
64 bytes from 172.19.0.2: seq=1 ttl=64 time=0.060 ms

--- alpine1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.046/0.053/0.060 ms
[root@docker ~]# docker exec alpine4 ping -c 2 alpine2
PING alpine2 (172.19.0.3): 56 data bytes
64 bytes from 172.19.0.3: seq=0 ttl=64 time=0.061 ms
64 bytes from 172.19.0.3: seq=1 ttl=64 time=0.083 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.061/0.072/0.083 ms
```

## 实验环境清理

删除所有容器及自建网络alpine-net

```bash
# docker container stop alpine1 alpine2 alpine3 alpine4

# docker container rm alpine1 alpine2 alpine3 alpine4

# docker network rm alpine-net
```




