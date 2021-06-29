# 实验 docker-007 host网络实验

查看主机网络链接

```bash
[root@docker ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:3d:1e:77 brd ff:ff:ff:ff:ff:ff
    inet 192.168.238.130/24 brd 192.168.238.255 scope global noprefixroute dynamic ens33
       valid_lft 1733sec preferred_lft 1733sec
    inet6 fe80::ab18:6b87:6ac2:1775/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
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
```

查看系统80，443端口是否被占用

```bash
[root@docker ~]# ss -antpl
```

没有占用的情况下，运行如下命令
```bash
[root@docker ~]# docker run --rm -d --network host --name nginx nginx   
```

通过宿主机IP地址访问nginx服务

![netowrk-host-nginx](./nginx-docker-host-netwok.png)
