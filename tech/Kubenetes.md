[toc]



# Docker

## 常用命令

```sh
docker build -t TAG .
docker run -v $PWD/data:docker_dir TAG

# Image
docker image ls // 列出镜像
docker image ls -a  // 包含中间层镜像
docker image ls -f dangling=true  // 包含悬虚镜像  -f == --filter
docker image prune  // 删除悬虚镜像
docker system df // 列出image, container, volume所占大小
docker image rm ID // 删除镜像
docker image rm $(docker image ls -q redis)  // 删除所有redis镜像
docker system prune -a

# Container
docker ps
docker container ls: 查看容器信息
docker container logs ID: 获取容器输出信息
docker container stop ID: 终止容器
docker system prune: 清理容器
docer exec -it $container /bin/sh: 进入容器

# 排查
docker events&
docker logs $instance_id

docker inspect $container
docker image inspect $image // 查看镜像 rootfs、...
```

docker exec 的原理

- setns() 系统调用
- 进程的每种 Linux Namespace 在对应的 `/proc/进程号/ns`下有一个虚拟文件，链接到一个真实的Namespace中
- 也就是说一个进程可以选择加入到某个进程已有的 Namespace 当中。



Dockerfile

```sh
# 使用官方提供的Python开发镜像作为基础镜像
FROM python:2.7-slim

# 将工作目录切换为/app
WORKDIR /app

# 将当前目录下的所有内容复制到/app下
ADD . /app

# 使用pip命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的80端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程
# 
# 等价于 docker run $image python app.py
# Docker 隐含 ENTRYPOINT: /bin/bash -c
# 所以等价于 /bin/sh -c "python app.py"
CMD ["python", "app.py"]
```





## 容器镜像 rootfs

- 容器镜像：挂载在容器根目录的文件系统，**rootfs** = /var/lib/docker/aufs/mnt
- 是容器的静态视图
- 注意容器镜像仅包含文件，不包含操作系统内核。--> 共享宿主机OS内核！



**Docker镜像的分层**

原理：联合文件系统 UnionFS, Union File System

- 只读层：ro + wh
  - readonly
  - whiteout 白障
  
- Init层：ro + wh
  - 位于只读层和读写层之间。专门永利来存放 /etc/hosts, /etc/resolv.conf 等信息。
  - 这些文件本来属于只读的 Ubuntu 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 hostname，所以就需要在可读写层对它们进行修改。但又不希望提交这些修改。
  
- 可读写层：rw
  - Read write
  - 用来存放修改 rootfs 后产生的增量：Copy On Write.
  - 如何删除文件：在可读写层创建一个 whiteout文件，把只读层里的文件遮挡起来。



**Volume**

用于将宿主机上的目录挂载到容器里面进行读取和修改操作。

原理：

- 容器镜像的各个层保存在 `/var/lib/docker/aufs/diff` 目录下；容器进程启动后，被联合挂载到 `/var/lib/docker/aufs/mnt` 目录。--> 这样 rootfs 就准备好了。

- 在 rootfs 准备好后、在 chroot 执行之前，把宿主机目录挂载到指定容器目录即可。例如 `/var/lib/docker/aufs/mnt/$可读写层ID/$dir`


```sh
docker volume ls

# 查看宿主机对应临时目录：同步修改
ls /var/lib/docker/volumes/{ID}/_data

# 查看宿主机可读写层：不会修改，所以 docker commit 时不会提交 volume 内容
ls /var/lib/docker/aufs/mnt/{ID}/{image dir}
```



## 容器运行时

- 容器运行时：由 Namespace + Cgroups 构成的隔离环境
- 是容器的动态视图



### Namespace

- 作用：**隔离**。例如容器内部 ps，只能看到该容器内的进程
- PID Namespace: 
  - 对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号，比如 PID=1。可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。
- Mount Namespace
  - 原理：**chroot**
    例如使用 $HOME/test 目录作为 /bin/bash 进程的根目录：`$ chroot $HOME/test /bin/bash`
- UTS Namespace
- Network Namespace
- User Namespace



### Cgroups

- Linux Control Group
- 作用：**限制**一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。
- 原理：
  - 以文件形式组织在 /sys/fs/cgroup
  - 控制组：/sys/fs/cgroup/xx
  - 子目录：cpu, blkio, cpuset, memory







# K8S 架构



作为开发者，并不需要关心容器运行时的差异。--> 催生”容器编排“，需求：

- 拉取镜像、运行容器
- 运维能力：路由网关、水平扩展、监控、备份、灾难恢复
- **处理大规模集群各种任务之间的关系。！！！**
  - 细粒度：分别做成镜像、运行在一个个专属的容器中，互不干涉，可以被调度在集群内任何一台机器上。
  - Docker compose? --> 方案太过简单
    - 例如能处理 Java Web + Mysql，但不能处理 Cassandra集群。
  - K8S 的思路：从更宏观角度，以统一的方式来定义任务之间的各种关系
    - 对容器的访问进行分类：
    - **Pod** 里的容器关系紧密：共享 network ns、volume；
    - **Service** 之间关系隔离：作为 pod的代理入口，维护 pod的ip、port等信息的自动更新
    - **Secret** 处理授权关系：pod启动时，以volume方式挂载secret里的数据





## Master 节点

kube-apiserver

- 负责 API 服务
- 处理集群的持久化数据，保存到 etcd



kube-schedule

- 负责调度



Kube-controller-manager

- 负责容器编排



## Node 节点

kubelet

- 负责和 `容器运行时` 打交道

  - 通过 CRI (Container Runtime Interface) 远程调用接口
  - 具体的容器运行时，例如Docker，则通过 OCI 规范同底层OS进行交互：把 CRI 请求翻译成对OS的系统调用

  

- 负责 通过 gRPC 和 `Device Plugin` 插件交互

  - 管理宿主机物理设备，例如 GPU
  - 主要用于通过k8s进行机器学习训练、高性能作业支持



## K8S 部署

**Step 1. 安装 kubeadm、kubelet、kubectl**

```sh

$ apt-get install kubeadm
```



- 把 kubelet 直接运行在宿主机上，然后使用容器部署其他 k8s 组件。

  - 为何 kubelet 不能用 容器部署？
    --> kubelet 要操作宿主机网络、数据卷

  

**Step 2. kubeadm init  部署 Master 节点** 

```sh
# 1. 创建 master 节点
kubeadm int --config kubeadm.yaml
# 查看节点
kubectl get nodes
# 查看pods: STATUS = PENDING
kubectl get pods -n kube-system 

# 2. 部署网络插件: 新增一个 pod，并使 CoreDNS, kube-controller-manager等依赖网络的Pod STATUS = Running
kubectl apply -f https://git.io/weave-kube-1.6

```



kubeadm init 工作流程

- **Preflight checks** 检查：os版本、cgroups模块是否可用、...
- 生成 k8s 对外提供服务所需的各种**证书和对应目录**
- 为其他组件生成访问 kube-apiserver 所需的**配置文件**：`/etc/kubernetes/xxx.conf`
- 为 master 组件 生成 **pod 配置文件**；
  - master 组件：kube-apiserver, kube-controller-manager, kube-scheduler
  - Static Pod: kubelet 启动时会检查 static pod yaml 文件目录 `/etc/kubernetes/manifests`，然后在这台机器上启动他们。 
- 为集群生成一个 **bootstrap token**
- 将 ca.crt 等 master 节点信息，通过 ConfigMap (cluster-info) 的方式保存到 Etcd，供后续部署 Node 节点使用。
- 安装默认插件
  - Kube-proxy
  - DNS



**Step 3. kubeadm join 部署 Worker 节点**

```sh
# 加入一个 node 节点
kubeadm join {master ip/port}
```



kubeadm join 工作流程

- 需要 bootstrap token，发起一次 ”不安全模式“ 的访问到 kube-apiserver，拿到 ConfigMap 中的 cluster-info (包含 apiserver 的授权信息)
- 以后即可以 ”安全模式“ 连接 apiserver



**Step 4. 部署 Dashboard 可视化插件**

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml
```



**Step 5. 部署存储插件**

存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。这样，无论你在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。这就是“持久化”的含义。

- Ceph - Rook
- GlusterFS
- NFS



# Kubectl 命令

**yaml 运行**

```sh
$ kubectl create -f xx.yaml
$ kubectl replace -f xx.yaml
$ kubectl apply -f xx.yaml # 推荐

$ kubectl get pods -l app=nginx
```



debug

```sh
$ kubectl describe pod xxx -n NAMESPACE

$ kubectl exec -it POD_NAME -- /bin/bash
```



# API Object 

## yaml 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

- kind
  - API 对象的类型
- metadata  
  - API 对象的标识
  - name
  - labels
  - annotations: 内部信息，用户不关心，k8s 组件本身会关注
- spec
  - 定义 Pod 模板：`spec.template`
    - 定义 `spec.template.metadata.lables`
    - 定义 容器镜像：`spec.template.spec.containers`
  - 定义 Label Selector: `spec.selector.matchLabels`





## Deployment

定义：是一个定义多副本应用的对象，即定义多个副本Pod；同时负责在 Pod 定义发生变化时对每个副本进行 Rolling Update.

> **控制器模式**：通过一个 API 对象管理另一个 API 对象；例如通过 Deployment 管理 Pod.



## Pod

定义：是k8s里最小的 API 对象，可以等价为一个应用（app，虚拟机），可以包含多个**紧密协作**的容器（container，用户程序）。

> **紧密协作**：类似 进程 vs. 进程组



**Pod 的作用**

- 便于调度：有亲密关系的容器调度到同一个node

- [容器设计模式](https://www.usenix.org/conference/hotcloud16/workshop-program/presentation/burns)：当你想在一个容器里跑多个功能不相关的应用时，应该优先考虑他们是不是更应该被描述成一个 Pod 里的多个容器。

  - 例1：War 包 + Tomcat

    > War: 定义为 Init Container，作为 sidecar 辅助容器，负责将war拷贝到指定 volume 目录；
    > Tomcat: 将相应 volume 目录挂载到 webapps 目录

  - 例2：容器的日志收集

    > 主容器写入 /var/log；
    >
    > Sidecar 容器不断读取 /var/log，转发到 ES



**Pod 的实现原理**

- Pod 是一个逻辑概念：k8s 真正处理的还是 namespace, cgroups
- Pod 里的所有容器，共享同一个 Network Namespace，可以声明共享同一个 Volume：原理是 关联同一个 “Infra 容器”
- 

















