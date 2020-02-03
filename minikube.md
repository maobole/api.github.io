# 配置代理

> 类似K8S，但有如下不同，重点是代理IP不能用127.0.0.1，设置的时候尽可能都用正式IP

## 配置Privoxy
```yml
listen-address 0.0.0.0:8118             # 要用0.0.0.0，也就是监听所有IP
forward-socks5t / 127.0.0.1:1080 .      # 转发到本地端口，注意最后有个点
```

### 设置系统代理
```bash
PROXY_HOST=172.16.144.129      # 要用主机的IP，不要用127.0.0.1，要让VM里能够访问
export all_proxy=http://$PROXY_HOST:8118
export ftp_proxy=http://$PROXY_HOST:8118
export http_proxy=http://$PROXY_HOST:8118
export https_proxy=http://$PROXY_HOST:8118
export no_proxy=192.168.100.60,192.168.100.61,192.168.100.62,192.168.0.0/24,10.244.0.0/16,10.96.0.0/12,127.0.0.1,172.17.0.0/16,localhost
```

# 安装minikube

## 下载minikube和kubectl
```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

## 安装KVM2
```bash
sudo yum -y install libvirt-daemon-kvm qemu-kvm libvirt virt-install bridge-utils
systemctl start libvirtd
systemctl enable libvirtd

sudo usermod -a -G libvirt $(whoami)

newgrp libvirt

curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-kvm2 && chmod +x docker-machine-driver-kvm2 && sudo mv docker-machine-driver-kvm2 /usr/bin/
```

## 启动minikube
```bash
minikube start --docker-env http_proxy=$http_proxy --docker-env https_proxy=$https_proxy --vm-driver=none
或
minikube start --docker-env http_proxy=$http_proxy --docker-env https_proxy=$https_proxy --vm-driver=kvm2

kubectl config use-context minikube

export no_proxy=$no_proxy,$(minikube ip)
export NO_PROXY=$no_proxy,$(minikube ip)

kubectl get pods --all-namespaces
```

## Build Image
### Docker Daemon
vi /etc/docker/daemon.json

```json
{
    "debug": true,
    "registry-mirrors": ["https://wka3w7l5.mirror.aliyuncs.com"],
    "hosts": [
        "tcp://0.0.0.0:2375",
        "unix:///var/run/docker.sock"
    ],
    "dns": [
        "114.114.114.114",
        "8.8.8.8"
    ],
    "insecure-registries": [
        "docker.lechat.io:8602"
    ]
}
```

## Usual Command
> 开放端口给外部访问
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'

> 强制替换部署
kubectl replace --force -f deploy.yaml

> 暂停节点
kubectl drain $NODENAME
>恢复节点
kubectl uncordon $NODENAME

> 非Master节点管理集群
scp /etc/kubernetes/admin.conf root@10.5.30.82:/etc/kubernetes/
export KUBECONFIG=/etc/kubernetes/admin.conf

> 强制删除处理Terminating状态的Pod两种方法
1. kubectl delete pod {pod name} --grace-period=0 --force
2. SSH into the node the stuck pod was scheduled on 
   Running docker ps | grep {pod name} to get the Docker Container ID
   Running docker rm -f {container id}

> 创建ConfigMap
kubectl create configmap config-files --from-file=/root/lechat-ali/configs

> 私有镜像仓库
kubectl create secret docker-registry docker-key --docker-server=docker.lechat.io:8602 --docker-username=docker --docker-password=jieyun@123 --docker-email=docker@lechat.io

