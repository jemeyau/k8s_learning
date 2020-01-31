### 1. 安装kubelet kubeadm kubectl和docker
需要设置阿里云的安装源
```shell
1. apt-get update && apt-get install -y apt-transport-https
2. curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
3. 
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
4. apt-get update
5. apt-get install -y kubelet kubeadm kubectl docker.io
```

### 2. kubeadm init
拉取kubeadm初始化需要的镜像，由于k8s.gcr.io国内无法访问，所以先通过阿里云的镜像手动拉取所需镜像,
具体需要哪些版本的镜像，可通过如下命令获取：
```shell
kubeadm config images list
```
[pull_k8s_images.sh](scripts/pull_k8s_images.sh)
```shell
#!/bin/sh
images=(  # 下面的镜像应该去除"k8s.gcr.io/"的前缀，版本换成上面获取到的版本
    kube-apiserver:v1.17.2
    kube-controller-manager:v1.17.2
    kube-scheduler:v1.17.2
    kube-proxy:v1.17.2
    pause:3.1
    etcd:3.4.3-0
    coredns:1.6.5
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

然后执行**kubeadm init**完成初始化流程。

别忘了执行如下命令，需要这些配置命令的原因是：Kubernetes 集群默认需要加密方式访问。所以，这几条命令，就是将刚刚部署生成的 Kubernetes 集群的安全配置文件，保存到当前用户的.kube 目录下，kubectl 默认会使用这个目录下的授权信息访问 Kubernetes 集群。
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

此时可以检查master节点的状态，应该为**NotReady**，因为尚未部署网络插件。
```shell
kubectl get nodes
```

### 3. 安装网络插件
执行如下命令即可
```shell
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

此时再检查节点状态，应该为**Ready**。

### 4. 安装dashboard
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
检查状态
```shell
kubectl get pods -n kube-system
```

Dashboard Service 添加
```shell
nodePort: 30001
```

开启代理
```shell
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'  --disable-filter=true &
```

获取token
```shell
kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
```

访问**https://<node-ip>:<node-port>**

### 5. 安装Rook
```shell
git clone https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl apply -f common.yaml
kubectl apply -f operator.yaml
kubectl apply -f cluster.yaml
```
检查状态
```shell
kubectl get pods -n rook-ceph-system
kubectl get pods -n rook-ceph
```
