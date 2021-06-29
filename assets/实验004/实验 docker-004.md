实验 docker-004 构建和使用私有仓库

前提：
1. docker正常运行；
2. 可访问dockerhub。



运行私有仓库

```bash
docker run -d -p 5000:5000 --restart always --name registry registry:2
```

修改docker daemon配置文件


为已有镜像修改tag，上传私有仓库


清理系统内镜像，从私有仓库拉去

link: https://docs.docker.com/registry/spec/api/



附加：
带有自签证书的仓库

自建CA：
```bash   
# 生成cakey.pem
# openssl genrsa -out ./private/cakey.pem 4096
```
自签证书：
```bash
# echo "01" > /etc/pki/CA/serial
# openssl req -new -x509 -key ./private/cakey.pem -out ./cacert.pem -days 36500
```
生成服务端私钥：
```bash
# 生成私钥
# openssl genrsa -out registrykey.pem 2048
# 生成证书请求
# openssl req -new -key registrykey.pem -out registry.csr -days 365
```        

向CA请求证书：
```bash
# openssl ca -in registry.csr out registry.crt -days 365
```

运行带有证书的regisrty：
```bash
docker run -d --restart=always \
--name registry -v "$(pwd)"/certs:/certs \
-e REGISTRY_HTTP_SECRET=true \
-e REGISTRY_HTTP_ADDR=0.0.0.0:443  \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registrycrt.pem  \
-e REGISTRY_HTTP_TLS_KEY=/certs/registrykey.pem \
-p 443:443 registry:2
```