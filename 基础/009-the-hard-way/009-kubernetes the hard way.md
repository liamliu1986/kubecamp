# Kubernetes The Hard Way

## 什么是RBAC

### Role和ClusterRole

- Role,指定namesapce
- ClusterRole

### RoleBinding和ClusterRoleBinding

## Cluster Details

高可用、端到端加密和RBAC认证的集群

- kubernetes
- containerd
- coredns
- cni
- etcd

## 安装工具

### 系统配置

关闭防火墙，关闭selinux

```bash
systemctl disable firewalld.service
setenforce 0
swapoff -a
```

下载二进制包

```bash
wget https://dl.k8s.io/v1.18.6/kubernetes-server-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.18.6/kubernetes-client-linux-amd64.tar.gz
yum install -y lrzs
```

### CFSSL

`packages/`目录下有cfssl和cfssljson的二进制程序，请在其中之一节点安装CFSSL。

### kubectl

请在master节点上分别安装`kubectl`

```bash
cp kubernetes/server/bin/kubectl /usr/local/bin/
```

配置免密访问

```bash
ssh-keygen -t rsa -b 4096
```

设置内核参数`net.ipv4.ip_forward=1`

## 自建CA和TSL证书

生成一个CA和一下组件的TLS证书：

- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kubelet
- kube-proxy

### 自建CA

```bash
cat > ca-config.json <<EOF
{
    "signing": {
        "default":{
            "expiry": "8760h"
        },
        "profiles": {
            "kubernetes": {
                "usages": ["signing", "key encipherment", "server auth", "client auth"],
                "expiry": "8760h"
            }
        }
    }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "neuedu",
      "OU": "CA",
      "ST": "LN"
    }
  ],
  "ca": {
    "expiry": "200000h"
  }
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### 客户端和服务器证书

#### 生成admin用户的client证书

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "system:masters",
      "OU": "neuedu",
      "ST": "LN"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json|cfssljson -bare admin
```

### kubelet client证书

```bash
cat > worker-csr.json <<EOF
{
  "CN": "system:node",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "system:nodes",
      "OU": "neuedu",
      "ST": "LN"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=kube-worker,192.168.92.161,kube-master,192.168.92.160 \
  -profile=kubernetes \
  worker-csr.json | cfssljson -bare kube-worker
```

### controller manager 证书

```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "system:kube-controller-manager",
      "OU": "neuedu",
      "ST": "LN"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

### kube proxy客户端证书

```bash
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "system:kube-proxy",
      "OU": "neuedu",
      "ST": "LN"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

### kube-scheduler客户端证书

```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "system:kube-scheduler",
      "OU": "neuedu",
      "ST": "LN"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
  ```

### kube-apiserver服务端证书

```bash
KUBERNETES_PUBLIC_ADDRESS=192.168.92.160,10.0.0.1
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "Kubernetes",
      "OU": "neuedu",
      "ST": "LN"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca/ca.pem \
  -ca-key=ca/ca-key.pem \
  -config=ca/ca-config.json \
  -hostname=kube-master,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

### 生成service account密钥对

```bash
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "DL",
      "O": "Kubernetes",
      "OU": "neuedu",
      "ST": "LN"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

### 分发证书

```bash
scp ca/ca.pem kube-worker.pem kube-worker-key.pem kube-worker:~/

cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem kube-worker.pem kube-worker-key.pem ~/
```

## 生成Kubernetes认证配置文件

### 客户端认证配置

生成`controller manager`，`kubelet`，`kube-proxy`，`scheduler`和`admin`用户

```bash
KUBERNETES_PUBLIC_ADDRESS=192.168.92.160
```

#### 生成admin配置文件

```bash
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca/ca.pem \
    --embed-certs=true \
    --server=https://192.168.92.160:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=ca/admin.pem \
    --client-key=ca/admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
```

#### 生成kuber-proxy配置文件

```bash
 kubectl config set-cluster kubernetes \
    --certificate-authority=ca/ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=ca/kube-proxy.pem \
    --client-key=ca/kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

#### 生成controller-manager配置文件

```bash
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca/ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=ca/kube-controller-manager.pem \
    --client-key=ca/kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
  ```

#### 生成kube-scheduler配置文件

```bash
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca/ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=ca/kube-scheduler.pem \
    --client-key=ca/kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

### 分发配置文件

```bash
scp kube-proxy.kubeconfig kube-worker:~/

cp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ~/
```

## 生成数据加密配置和key

### 加密秘钥

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/random | base64)
```

### 加密配置文件

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

### 分发秘钥配置文件

```bash
cp encryption-config.yaml ~/
```

## 启动一个etcd集群


### 启动一个etcd集群成员

```bash
wget "https://github.com/etcd-io/etcd/releases/download/v3.4.10/etcd-v3.4.10-linux-amd64.tar.gz"

tar xzvf etcd-v3.4.10-linux-amd64.tar.gz
mv etcd-v3.4.10-linux-amd64/etcd* /usr/local/bin/

mkdir -pv /etc/etcd /var/lib/etcd
chmod 700 /var/lib/etcd
cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/

INTERNAL_IP=192.168.92.160
ETCD_NAME=kube-master

```

生成etcd.service

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster kube-master=https://192.168.92.160:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

启动服务

```bash
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

验证

```bash
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

## 初始化控制节点

创建配置文件目录

```bash
mkdir -pv /etc/kubernetes/config
```

下载相关二进制文件

```bash
kube-apiserver
kube-controller-manager
kube-scheduler
kubectl

mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### 配置kubernetes API Server

```bash
mkdir -pv /var/lib/kubernetes

mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem encryption-config.yaml /var/lib/kubernetes

INTERNAL_IP=192.168.92.160

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=1 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.92.160:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 配置kubernetes controller-manager

```bash
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=false \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 配置kubernetes scheduler

```bash
mv kube-scheduler.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --leader-elect=false \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 启动所有服务

```bash
systemctl daemon-reload
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

#### 生成kubelet客户配置文件

使用admin.kubeconfig作为kubelet的配置文件

<!-- ***这里有权限问题，使用admin.kubeconfig替代，但是可以自定clusterrole***

```bash
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca/ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kubelet.kubeconfig

  kubectl config set-credentials system:node  \
    --client-certificate=ca/kube-worker.pem \
    --client-key=ca/kube-worker-key.pem \
    --embed-certs=true \
    --kubeconfig=kubelet.kubeconfig 

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:node \
    --kubeconfig=kubelet.kubeconfig

  kubectl config use-context default --kubeconfig=kubelet.kubeconfig
``` -->

### 验证

```bash
[root@controller01 ~]# kubectl get componentstatuses --kubeconfig admin.kubeconfig
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```

<!-- ### kubelet的RBAC认证

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
``` -->

验证

```bash
[root@controller01 ~]# curl --cacert /var/lib/kubernetes/ca.pem https://127.0.0.1:6443/version
{
  "major": "1",
  "minor": "19",
  "gitVersion": "v1.19.0",
  "gitCommit": "e19964183377d0ec2052d1f1fa930c4d7575bd50",
  "gitTreeState": "clean",
  "buildDate": "2020-08-26T14:23:04Z",
  "goVersion": "go1.15",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

## 部署worker节点

### 准备worker节点

关闭swap

```bash
[root@worker01 ~]# swapoff -a
```

创建指定目录

```bash
[root@worker01 ~]# mkdir -p \
/etc/cni/net.d \
/opt/cni/bin \
/var/lib/kubelet \
/var/lib/kube-proxy \
/var/lib/kubernetes \
/var/run/kubernetes
```

解压缩CNI插件，安装其他可执行程序

```bash
[root@worker01 ~]# tar xvf cni-plugins-linux-amd64-v0.8.7.tgz -C /opt/cni/bin/
[root@worker01 ~]# chmod +x kubectl kubelet kube-proxy
[root@worker01 ~]# mv kubectl kubelet kube-proxy /usr/local/bin/
```

设置CNI网络，必须与master上配置的CIDR一致

```bash
[root@worker01 ~]# POD_CIDR=10.200.0.0/16
```

部署kubernetes相关配置文件

```bash
[root@kube-master ~]# mv kube-worker-key.pem kube-worker.pem /var/lib/kubelet/
[root@kube-master ~]# mv kubelet.kubeconfig /var/lib/kubelet/
[root@worker01 ~]# mv ca.pem /var/lib/kubernetes/
```

创建kubelet配置文件

```bash
[root@worker01 ~]# cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kube-worker.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/kube-worker-key.pem"
EOF
```

配置kubelet服务

```bash
[root@worker01 ~]# cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime=docker \
  --image-pull-progress-deadline=2m \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --network-plugin=cni \
  --register-node=true \
  --v=2
Restart=on-failure
RestartSec=5
 
[Install]
WantedBy=multi-user.target
EOF
```

配置kube-proxy配置文件

```bash
[root@worker01 ~]# mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

创建运行配置文件

```bash
[root@worker01 ~]# cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

创建kube-proxy启动文件

```bash
[root@worker01 ~]# cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

启动服务

```bash
[root@worker01 ~]# systemctl enable  kubelet kube-proxy
[root@worker01 ~]# systemctl start  kubelet kube-proxy
```

创建桥接网络配置文件

```bash
[root@worker01 ~]# cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.200.1.0/24"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

创建loopback网络配置文件

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
```

测试Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - name: http-nginx
          containerPort: 80
          protocol: TCP
```


