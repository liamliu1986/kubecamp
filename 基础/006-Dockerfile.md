# Dockerfile

## Dockerfile
Docker从基础设施文件Dockerfile自动构建images。

Dockerfile是一个包含构建镜像所需的所有命令的有序的文本文件。

docker images有多个只读layer组成，每个layer就是Dockerfile中的一个指令。

```dockerfile
FROM ubuntu:18.04
COPY . /app
RUN  make /app
CMD  python /app/app.py
```

## 格式

```bash
INSTRUCTION arguments
```

### FROM

```dockerfile
FROM [--platform=<platform>] <image> [AS <name>]
# or
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
# or
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```


### RUN

```dockerfile
RUN <command>  

RUN ["executalbe", "param1", "param2"]  #不调用shell
```
在新建layer上执行指定命令，将结构构建为新层，作为下一步骤的基础层

默认情况会缓存当前的构建，`--no-cache`来取消缓存


### CMD

```dockerfile
CMD ["executable","param1","param2"]

CMD ["param1","param2"]            # 为entrypoint提供参数

CMD command param1 param2
```

### LABEL

添加metadata

### EXPOSE

```dockerfile
EXPOSE <port> [<port>/<protocol>]
```
不实际发布接口，仅仅是一种声明


### ENV

```dockerfile
ENV <key>=<value> ...
```


### ADD

```dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

ADD指令的使用原则：
- <src>必须再构建上下文路径中
- 如果<src>是一个URL，并且<dest>不以'/'结束，将从src下载一个文件copy到<dest>
- 如果<src>是一个URL，并且<dest>以'/'结束，将src下载一个文件copy为<dest>/<filename>
- <src>是一个目录，拷贝将包含目录全部内容和元数据
- 可识别压缩归档文件将被解压为目录
- 其他格式文件<src>, <dest>以'/'结尾，ADD执行结果为<dest>/<src>
- 多<src>，则<dest>必须为目录
- 如果<dest>不以'/', <src>将被写入<dest>(视为文件)
- <dest>不存在将会被创建


### COPY

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

与ADD差异，不支持压缩包解压缩的功能


### ENTRYPOINT

```dockerfile
ENTRYPOINT ["executable", "param1", "param2"]

ENTRYPOINT command param1 param2
```

### VOLUME


### USER
```dockerfile
USER <user>[:<group>]

USER <UID>[:<GID>]
```
指定容器运行、RUN、CMD、ENTRYPOINT的属主


### WORKDIR

指定工作目录

### ARG

为build过程提供变量支持
```dockerfile
ARG <name>[=<default value>]
```

在docker build中可使用--build-arg \<varname>=\<value>，传递给dockerfile


### ONBUILD

```dockerfile
ONBUILD INSTRUCTION arguments
```
这些指令将被缓存，当该image作为其他镜像的base时，首先会按照顺序执行这些指令


### HEALTHCHECK

```dockerfile
HEALTHCHECK [OPTIONS] CMD command
```

可用参数
- --interval=DURATION(30s)
- --timeout=DURATION(30s)
- --start-period=DURATION(0s)
- --retries=N(3)

### SHELL
修改默认的shell
```dockerfile
SHELL ["/bin/sh", "-c"]

SHELL ["cmd", "/S", "/C"]
```

## 最佳实践

### 1. 理解构建上下文

***'.' 很重要***

```dockerfile 
docker build [OPTIONS] -f- PATH << EOF
...
...
EOF
```

### 2. .dockerignore

### 3. 多阶段构建

多阶段构建可以在无需减少中间层的情况下，大幅降低image的最终大小。

控制images大小是镜像构建的一个重要挑战，每条指令在image中增加layer，你需要在构建下一个layer前，清理所有不相关的层级结构。这还涉及到很多shell相关的技巧，来实现控制目标。

***示例：***

`Dockerfile.build`

```dockerfile
FROM golang:1.7.3
WORKDIR /go/src/github.com/alexellis/href-counter/
COPY app.go .
RUN go get -d -v golang.org/x/net/html \
  && CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
```

`Dockerfile`

```dockerfile
FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY app .
CMD ["./app"]  
```

`build.sh`

```bash
#!/bin/sh
echo Building alexellis2/href-counter:build

docker build --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy \  
    -t alexellis2/href-counter:build . -f Dockerfile.build

docker container create --name extract alexellis2/href-counter:build  
docker container cp extract:/go/src/github.com/alexellis/href-counter/app ./app  
docker container rm -f extract

echo Building alexellis2/href-counter:latest

docker build --no-cache -t alexellis2/href-counter:latest .
rm ./app
```

***多阶段构建***

无需build.sh，直接一条构建命令即可实现上述效果

```dockerfile
FROM golang:1.7.3
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html  
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]  
```

### 4. 不要安装非必要的package

### 5. 隔离应用

### 6. 最小化layers

### 7. 排序多行变量

***Example***

```dockerfile
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion \
  && rm -rf /var/lib/apt/lists/*
```

### 8. 利用构建缓存

构建过程中docker会先查找构建缓存， 通过`--no-cache=true`可以在构建过程中禁止使用缓存

原则：

- 构建过程中搜索缓存是否存在相同layer，存在直接调用缓存，不存在构建新layer
- 任意文件变更缓存都将失效，不考虑部分元数据
- 除文件相关命令外，只检查指令字符串匹配