实验 docker-003 使用docker commit生成自定义镜像


前提：
1. docker正常运行；
2. 可访问dockerhub。


运行一个centos container

```bash
docker run -itd centos sleep 3000
```

登录容器，执行：

```bash
yum install -y vim
```

退出容器ctrl+d，执行：
```bash
docker commit <container name>  <image name>:[tag]


docker history <image name>:[tag]
```

仔细观察，发生了什么？





