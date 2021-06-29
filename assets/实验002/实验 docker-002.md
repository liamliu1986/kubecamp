实验 docker-002 共享内核分析

前提：
1. docker正常运行；
2. 可访问dockerhub。


查看系统内核版本
```bash
uname -r
```

创建centos容器，查看内核版本
```bash
docker run --rm --name centos centos uname -r
```

创建busybox容器，查看内核版本
```bash
docker run --rm --name busybox busybox uname -r
```

关注容器生命周期


查看独立的rootfs
```bash
docker run --rm --name centos centos ls /
docker run --rm --name busybox busybox ls /
```




