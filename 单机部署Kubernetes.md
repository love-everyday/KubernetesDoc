# 单机部署Kubernetes

初学阶段，尝试在阿里云的Ubuntu云服务器上部署Kubernetes，并运行镜像Nginx。

## 环境

* 阿里云
* Ubuntu 16.0
* root权限

## 安装

### HTTPS安装传输软件包

```bash
apt-get update
apt-get install apt-transport-https ca-certificates curl software-properties-common
```

### 添加GPG密钥

```bash
curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```

向`source.list`中添加中科大的Docker源

```bash
add-apt-repository \
    "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
```

### 安装docker

```bash
apt-get update
apt-get install docker
```

### 启动Docker

```bash
systemctl enable docker
systemctl start docker
```

### 关闭Swap

```bash
swapoff -a
```

### 使用阿里云的源安装Kubeadm

```bash
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
```

### 修改`kubelet`的`cgroup`

* 查看Docker的cgroup

```bash
docker info | grep -i cgroup
```

* 如果返回是`Cgroup Driver: cgroupfs`，则需要修改，因为`kubelet`的`cgroup`为`system`，修改操作：

```bash
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

* 添加如下内容：

```bash
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
```

* 重启`kubelet`

```bash
systemctl daemon-reload
systemctl restart kubelet
```

### 安装Kubeadm启动需要的镜像

* 获取Kubeadm启动需要的镜像

```bash
kubeadm config images list
```

* 返回的需要安装的镜像，如下：

```bash
k8s.gcr.io/kube-apiserver:v1.13.3
k8s.gcr.io/kube-controller-manager:v1.13.3
k8s.gcr.io/kube-scheduler:v1.13.3
k8s.gcr.io/kube-proxy:v1.13.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.2.24
k8s.gcr.io/coredns:1.2.6
```

* 创建一个脚本，将谷歌的镜像替换成阿里云的镜像：

```bash
vi kubeadm-aliyun.sh
```

* 在`kubeadm-aliyun.sh`脚本中，添加如下内容：

```bash
#!/bin/bash
images=(
    kube-apiserver:v1.13.3
    kube-controller-manager:v1.13.3
    kube-scheduler:v1.13.3
    kube-proxy:v1.13.3
    pause:3.1
    etcd:3.2.24
    coredns:1.2.6
)
for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
done
```

* 赋予`kubeadm-aliyun.sh`脚本执行权限，并执行

```bash
chmod 777 kubeadm-aliyun.sh
bash kubeadm-aliyun.sh
```

### 初始化Kubeadm

```bash
kubeadm init --pod-network-cidr=192.168.0.0/16
```

* 初始化成功后，可以看到如下内容：

```bash
...

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

...
```

* 添加用户配置

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* 查看所有pod，执行命名`kubectl get pods --all-namespaces`，返回如下内容：

```bash
NAMESPACE     NAME                                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-86c58d9df4-bvtgh                          0/1     Pending   0          25m
kube-system   coredns-86c58d9df4-bx284                          0/1     Pending   0          25m
kube-system   etcd-izbp162mggaelsax3pz9n6z                      1/1     Running   0          24m
kube-system   kube-apiserver-izbp162mggaelsax3pz9n6z            1/1     Running   0          24m
kube-system   kube-controller-manager-izbp162mggaelsax3pz9n6z   1/1     Running   0          24m
kube-system   kube-proxy-xd4dv                                  1/1     Running   0          25m
kube-system   kube-scheduler-izbp162mggaelsax3pz9n6z            1/1     Running   0          24m
```

可以看到`coredns`是`Pending`状态，这是因为还需要安装网络插件

### 安装网络插件`Calico`

* 查看[官网](https://docs.projectcalico.org/v3.5/getting-started/kubernetes/)，根据官方文档安装

```bash
kubectl apply -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/installation/hosted/etcd.yaml
kubectl apply -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/installation/hosted/calico.yaml
```

* 再次查看所有pod状态，`kubectl get pods --all-namespaces`

```bash
NAMESPACE     NAME                                              READY   STATUS    RESTARTS   AGE
kube-system   calico-etcd-6fb7s                                 1/1     Running   0          82s
kube-system   calico-kube-controllers-74887d7bdf-8gf9w          1/1     Running   0          98s
kube-system   calico-node-zxhfb                                 1/1     Running   2          97s
kube-system   coredns-86c58d9df4-bvtgh                          1/1     Running   0          34m
kube-system   coredns-86c58d9df4-bx284                          1/1     Running   0          34m
kube-system   etcd-izbp162mggaelsax3pz9n6z                      1/1     Running   0          33m
kube-system   kube-apiserver-izbp162mggaelsax3pz9n6z            1/1     Running   0          33m
kube-system   kube-controller-manager-izbp162mggaelsax3pz9n6z   1/1     Running   0          33m
kube-system   kube-proxy-xd4dv                                  1/1     Running   0          34m
kube-system   kube-scheduler-izbp162mggaelsax3pz9n6z            1/1     Running   0          33m
```

在多次刷新后，可以看到所有的pod的状态都是`Running`

## 部署Nginx镜像

* 测试环境下，解除`Master`限制，让`Pod`可以在`Master`上运行

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

* 创建`nginx`的deployment配置文件

```bash
vi nginx_deployment.yaml
```

* 在配置文件`nginx_deployment.yaml`中，添加如下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.0
        ports:
        - containerPort: 80
```

* 创建deployment

```bash
kubectl create -f nginx_deployment.yaml
```

* 查看pod，执行命令`kubectl get pods`，运行正常会返回如下内容：

```bash
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-78f64cd6dd-2tv74   1/1     Running   0          69s
nginx-deployment-78f64cd6dd-dbhz8   1/1     Running   0          69s
```

* 配置Service，让外部可以访问pod

```bash
vi nginx_service.yaml
```

* 在`nginx_service.yaml`中，添加配置信息：
  * `nodePort`，用来提供给Cluster外部客户访问service的端口
  * `port`，用来提供给Cluster内部客户访问service的端口
  * `targetPort`，pod的端口

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 9898
      targetPort: 80
      nodePort: 31000
  type: NodePort
```

* 创建Service

```bash
kubectl create -f nginx_service.yaml
```

* 查询Service

```bash
kubectl get svc
```

* 返回Service信息如下：

```bash
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP          94m
nginx-service   NodePort    10.107.117.32   <none>        9898:31000/TCP   11s
```

* 通过浏览器访问`<your-hostname>:31000`，如果部署成功就会返回nginx的`Welcome to nginx!`初始页面

## 参考资料

[Ubuntu上快速安装单机版Kubernetes](https://segmentfault.com/a/1190000016144782)  
[国内环境使用Kubeadm部署Kubernetes](https://juejin.im/post/5b8a4536e51d4538c545645c)  
[Quickstart for Calico on Kubernetes - Calico Doc](https://docs.projectcalico.org/v3.5/getting-started/kubernetes/)  
[Creating a single master cluster with kubeadm - Kubernetes Doc](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
[Deployment - Kubernetes Doc](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
