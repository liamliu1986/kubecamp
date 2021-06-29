# 实验 docker-010 使用volume
Volume是保存Docker容器生成和使用的数据的首选机制。bind mount依赖于主机的目录结构和操作系统，volume完全由Docker管理。卷与绑定挂载相比有几个优势:
- volume更容器备份和合并数据
- 可以通过Docker API或Docker CLI管理volumes
- volumes可以在linux或windows容器中工作
- 可以更安全的在多个容器中共享数据
- volume drivers提供远程存储或云存储的支持
- 新volume的内容可以通过容器填充
- volumes 比 docker desktop在Mac和Windows平台上使用bind mount具有更好的性能

**两种选项表示方式**

```bash
-v, --volume [volume name]:<container path>:[option,...]

--mount <key>=<value>,...
```

***当与service一起使用时，只有--mount有效***

## 创建volume

创建一个volume
```bash
[root@docker01 ~]# docker volume create my-vol
my-vol
[root@docker01 ~]# ll /var/lib/docker/volumes/
total 24
-rw------- 1 root root 32768 Nov 25 22:40 metadata.db
drwxr-xr-x 3 root root    19 Nov 25 22:40 my-vol

[root@docker01 ~]# docker volume ls
DRIVER              VOLUME NAME
local               my-vol

[root@docker01 ~]# docker volume inspect my-vol
[
    {
        "CreatedAt": "2020-11-25T22:40:37+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```
清理
```bash
[root@docker01 ~]# docker volume rm my-vol 
my-vol
[root@docker01 ~]# docker volume ls
DRIVER              VOLUME NAME
```

## 启动容器自动挂载volume

--mount

```bash
[root@docker01 ~]# docker run -d \
>   --name devtest \
>   --mount source=myvol2,target=/app \
>   nginx:latest

[root@docker01 ~]# ll /var/lib/docker/volumes/
total 24
-rw------- 1 root root 32768 Nov 25 23:18 metadata.db
drwxr-xr-x 3 root root    19 Nov 25 23:18 myvol2
```

-v

```bash
[root@docker01 ~]# docker run -d \
>   --name devtest \
>   -v myvol2:/app \
>   nginx:latest

[root@docker01 ~]# ll /var/lib/docker/volumes/
total 24
-rw------- 1 root root 32768 Nov 25 23:18 metadata.db
drwxr-xr-x 3 root root    19 Nov 25 23:18 myvol2
```

清理环境
```bash
[root@docker01 ~]# docker container stop devtest

[root@docker01 ~]# docker container rm devtest

[root@docker01 ~]# docker volume rm myvol2
```

当镜像中存在文件，这些文件将填充入volume

一下命令干了什么？
Tips: --volumes-from，将某容器的volume挂在给新的容器


```bash
$ docker run --rm --volumes-from dbstore -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata

$ docker run --rm --volumes-from dbstore2 -v $(pwd):/backup ubuntu bash -c "cd /dbdata && tar xvf /backup/backup.tar --strip 1"
```

bind mount 略