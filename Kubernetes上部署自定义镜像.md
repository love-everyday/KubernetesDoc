# 在Kubernetes上部署自定义仓库的镜像

在完成[Kubernetes单机部署](./单机部署Kubernetes.md)后，尝试在云服务器上`Deployment`自定义的Node.js应用镜像。

## 环境

* 已完成单机部署Kubernetes的Ubuntu服务器
* 在[阿里云镜像仓库](https://cr.console.aliyun.com)中有一份Node.js应用的镜像

## 部署

### 部署自定义镜像

* 创建`deployment_reactssr.yaml`配置文件，并添加以下配置内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reactssr-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: reactssr
  template:
    metadata:
      labels:
        app: reactssr
    spec:
      containers:
      - name: reactssr
        image: registry.cn-hangzhou.aliyuncs.com/仓库/镜像:版本
        ports:
        - containerPort: 3000
```

* 创建`reactssr-deployment`

```bash
kubectl create -f deployment_reactssr.yaml --record
```

* 运行命令`kubectl get pods`，会发现`reactssr-deployment`的状态是`ImagePullBackOff`，因为k8s拉取私有仓库的镜像失败，可以使用命令`kubectl describe pod  reactssr-deployment-7d6f59fc7d-mffsv`，返回以下内容：

```bash
...
Events:
  Type     Reason     Age                  From                              Message
  ----     ------     ----                 ----                              -------
  Normal   Scheduled  2m49s                default-scheduler                 Successfully assigned default/reactssr-deployment-7d6f59fc7d-mffsv to izbp162mggaelsax3pz9n6z
  Normal   BackOff    91s (x6 over 2m47s)  kubelet, izbp162mggaelsax3pz9n6z  Back-off pulling image "registry.cn-hangzhou.aliyuncs.com//仓库/镜像:版本"
  Normal   Pulling    79s (x4 over 2m48s)  kubelet, izbp162mggaelsax3pz9n6z  pulling image "registry.cn-hangzhou.aliyuncs.com//仓库/镜像:版本"
  Warning  Failed     79s (x4 over 2m48s)  kubelet, izbp162mggaelsax3pz9n6z  Failed to pull image "registry.cn-hangzhou.aliyuncs.com//仓库/镜像:版本": rpc error: code = Unknown desc = Error response from daemon: pull access denied for registry.cn-hangzhou.aliyuncs.com/仓库/镜像, repository does not exist or may require 'docker login'
  Warning  Failed     79s (x4 over 2m48s)  kubelet, izbp162mggaelsax3pz9n6z  Error: ErrImagePull
  Warning  Failed     67s (x7 over 2m47s)  kubelet, izbp162mggaelsax3pz9n6z  Error: ImagePullBackOff
```

* 使用`Kubernetes`的`Secret`来解决，拉取私有库镜像的验证问题

```bash
kubectl create secret docker-registry myregistrykey --docker-server={server} --docker-username={username} --docker-password={password}
```

* 重新编辑`reactssr-deployment`的配置，运行命令`kubectl edit deployment/reactssr-deployment`，添加`imagePullSecrets`。

* 创建配置文件`reactssr_service.yaml`。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: reactssr-service
spec:
  selector:
    app: reactssr
  ports:
    - protocol: TCP
      port: 9899
      targetPort: 3000
      nodePort: 31001
  type: NodePort
```

* 创建`Service`

```bash
kubectl create -f reactssr_service.yaml
```

## 参考资料

[Secret - Kubernetes中文 Doc](https://www.kubernetes.org.cn/secret)