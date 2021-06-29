# Macvaln实验

实用性较低

创建一个macvaln-net

```bash
[root@docker01 ~]# docker network create -d macvlan \
> --subnet=192.168.238.0/24 \
> --gateway=192.168.238.2 \
> -o parent=ens32 macvlan-net
2580b2c1e496fea2020497fe3eb5a34f11233e5afc232adc8270725d26eb3e6e
```
创建一个测试容器

```bash
[root@docker01 ~]# docker run --rm -itd --network macvlan-net --name macvlan-alpine alpine ash
d9a3622bf45e0ebd39cb5298bbf544e5e53d66b8aa91e9a117eb7009b1e6f56d
```

创建一个物理网络的子接口

```bash
[root@docker01 ~]# docker network create -d macvlan \
> --subnet=192.168.238.0/24 \
> --gateway=192.168.238.2 \
> -o parent=ens32.10 macvlan-net

[root@docker01 ~]# ip a
82: ens32.10@ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 00:0c:29:ca:0d:6c brd ff:ff:ff:ff:ff:ff
    inet6 fe80::20c:29ff:feca:d6c/64 scope link 
       valid_lft forever preferred_lft forever


[root@docker01 ~]# lsmod |grep 8021q
8021q                  33080  0 
garp                   14384  1 8021q
mrp                    18542  1 8021q
```