[toc]



# | Docker

## || 常用命令

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





## || 容器镜像: rootfs

- 容器镜像：挂载在容器根目录的文件系统，**rootfs** = /var/lib/docker/aufs/mnt
- 是容器的静态视图
- 注意容器镜像仅包含文件，不包含操作系统内核。--> 共享宿主机OS内核！



**Docker镜像的分层**

原理：联合文件系统 UnionFS (Union File System)

- **只读层：ro + wh**
  - readonly
  - whiteout 白障
  
- **Init 层：ro + wh**
  - 位于只读层和读写层之间。专门永利来存放 /etc/hosts, /etc/resolv.conf 等信息。
  - 这些文件本来属于只读的 Ubuntu 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 hostname，所以就需要在可读写层对它们进行修改。但又不希望提交这些修改。
  
- **可读写层：rw**
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



## || 容器运行时

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



### Cgroup

- Linux Control Group
- 作用：**限制**一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。
- 原理：
  - 以文件形式组织在 /sys/fs/cgroup
  - 控制组：/sys/fs/cgroup/xx
  - 子目录：cpu, blkio, cpuset, memory







# | K8S 架构



作为开发者，并不需要关心容器运行时的差异。--> 催生”容器编排“，需求：

- 拉取镜像、运行容器
- 运维能力：路由网关、水平扩展、监控、备份、灾难恢复
- **处理大规模集群各种任务之间的关系。！！！**
  - 细粒度：分别做成镜像、运行在一个个专属的容器中，互不干涉，可以被调度在集群内任何一台机器上。
  - Docker compose? --> 方案太过简单
    - 例如能处理 Java Web + Mysql，但不能处理 Cassandra 集群。
  - K8S 的思路：从更宏观角度，以统一的方式来定义任务之间的各种关系
    - 对容器的访问进行分类：
    - **Pod** 里的容器关系紧密：共享 network ns、volume；
    - **Service** 之间关系隔离：作为 pod 的代理入口，维护 pod 的 ip、port 等信息的自动更新
    - **Secret** 处理授权关系：pod 启动时，以 volume 方式挂载 secret 里的数据



## || Master 节点

**kube-apiserver**

- 负责 API 服务
- 处理集群的持久化数据，保存到 etcd



**kube-schedule**

- 负责调度



**kube-controller-manager**

- 负责容器编排



## || Node 节点

**kubelet**

- 负责和 `容器运行时` 打交道

  - 通过 CRI (Container Runtime Interface) 远程调用接口
  - 具体的容器运行时，例如Docker，则通过 OCI 规范同底层 OS 进行交互：把 CRI 请求翻译成对 OS 的系统调用

  

- 负责 通过 gRPC 和 `Device Plugin` 插件交互

  - 管理宿主机物理设备，例如 GPU
  - 主要用于通过 k8s 进行机器学习训练、高性能作业支持



## || K8S 部署

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



`kubeadm init` 工作流程

- **Preflight checks** 检查：os 版本、cgroups 模块是否可用、...
- 生成 k8s 对外提供服务所需的各种**证书和对应目录**
- 为其他组件生成访问 kube-apiserver 所需的**配置文件**：`/etc/kubernetes/xxx.conf`
- 为 master 组件生成 **pod 配置文件**；
  - master 组件：`kube-apiserver`, `kube-controller-manager`, `kube-scheduler`
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



# | Kubectl 命令

yaml 运行

```sh
$ kubectl create -f xx.yaml
$ kubectl replace -f xx.yaml
$ kubectl apply -f xx.yaml # 推荐

$ kubectl get pods -l app=nginx
```



debug

```sh
# 查看
$ kubectl describe pod xxx -n NAMESPACE
# 进入
$ kubectl exec -it POD_NAME -- /bin/bash
```



运维

```sh
# 扩缩容
$ kubectl scale deployment nginx-deployment --replicas=4

# 查看 deployment 对象的状态变化
$ kubectl rollout status deployment/nginx-deployment

# 编辑 etcd 里的 API对象
$ kubectl edit deployment/nginx-deployment

# set image: --record 记录到 rollout history
$ kubectl set image deployment/xx yyy --record

# 部署历史
$ kubectl rollout history deployment/xx
$ kubectl rollout history deployment/xx --revision=2 

# 部署回滚
$ kubectl rollout undo deployment/xx
$ kubectl rollout undo deployment/xx --to-revision=2
```





# | API Object 

## || yaml 配置

```yaml
apiVersion: apps/v1
#API 对象的类型
kind: Deployment 
#API 对象的标识: name, labels, annotations (内部信息，用户不关心，k8s 组件本身会关注)
metadata:
  name: nginx-deployment
spec:
  # 定义 Label Seclector
  selector:
    matchLabels: 
      app: nginx
  replicas: 2
  # 定义 Pod 模板：`spec.template`
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers: ##模板：容器镜像
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```



## || Pod

**定义：**

是 k8s 里最小的 API 对象，可以等价为一个应用（`app`，虚拟机），可以包含多个**紧密协作**的容器（`container`，用户程序）。

> **紧密协作**：类似 进程 vs. 进程组

- Pod: 类似虚拟机
- Container: 类似用户程序



**作用**

- 便于调度：有亲密关系的容器调度到同一个 node

- [容器设计模式](https://www.usenix.org/conference/hotcloud16/workshop-program/presentation/burns)：当你想在一个容器里跑多个功能不相关的应用时，应该优先考虑他们是不是更应该被描述成一个 Pod 里的多个容器。

  - 例1：War 包 + Tomcat

    > War: 定义为 Init Container，作为 sidecar 辅助容器，负责将 war 拷贝到指定 volume 目录；
    > Tomcat: 将相应 volume 目录挂载到 webapps 目录

  - 例2：容器的日志收集

    > 主容器写入 /var/log；
    >
    > Sidecar 容器不断读取 /var/log，转发到 ES



**实现原理**

- Pod 是一个逻辑概念：k8s 真正处理的还是 namespace, cgroups
- Pod 里的所有容器，共享同一个 Network Namespace，可以声明共享同一个 Volume：原理是 关联同一个 “Infra 容器”



**生命周期**

- **Pending**: yaml 文件已经提交给 k8s，API 对象已被创建 并保存在 Etcd 中。
- **Running**：调度成功，并跟一个具体节点绑定。
- **Scceeded**：所有容器运行完毕并退出。常见于运行一次性任务。
- **Failed**：有容器以非 0 状态码退出。
- **Unknown**：Pod状态不能被 `kubelet` 汇报给 `kube-apiserver`，可能是主从节点通信问题。



**Pod Yaml 字段**

*Pod Level* 

- `nodeSelector`：用于将 Pod 与 Node 进行绑定。下例表示这个pod 只能运行在有 `disktype: ssd` label 的节点上。

  ```yaml
  apiVersion: v1
  kind: Pod
  ...
  spec:
   nodeSelector:
     disktype: ssd
  ```

  

- `nodeName`：表示该Pod 已经经过了调度。用户可设置它来骗过调度器。

- `hostAliases`：定义 Pod 的 /etc/hosts

- `restartPolicy`: 

  - Always：只要容器不再运行状态，就自动重启。--> Deployment 控制的 Pod 重启策略只能是 Always.
  - OnFailure：只有在容器异常时才自动重启。
  - Never：适合关注容器退出后的日志、文件，或批处理任务Job

*Container Level*

- `imagePullPolicy`：Always | Never

- `lifecycle`: postStart, preStop 时触发一系列钩子

- `readinessProbe`：决定该 pod 能否通过 Service 的方式访问到；

  ```yaml
          #Liveness
          livenessProbe:
            exec:
              command: ["mysqladmin", "ping"]
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          #Rediness  
          readinessProbe:
            exec:
              # 通过TCP连接的方式进行健康检查
              command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
            initialDelaySeconds: 5
            periodSeconds: 2
            timeoutSeconds: 1
  ```

  

- `livenessProbe` : 健康检查探针

  ```yaml
  livenessProbe:
       httpGet:
         path: /healthz
         port: 8080
         httpHeaders:
         - name: X-Custom-Header
           value: xx
         initialDelaySeconds: 3
         periodSeconds: 3
  ```



## || Deployment / ReplicaSet

**定义**：

是一个定义多副本应用的对象，即定义多个副本 Pod；同时负责在 Pod 定义发生变化时对每个副本进行 Rolling Update。

> **控制器模式**：通过一个 API 对象管理另一个 API 对象；例如通过 Deployment 管理 Pod。



**作用：**

- Pod 水平扩缩容 （horizontal scaling out / in）

  ```sh
  $ kubectl scale deployment nginx-deployment --replicas=4
  ```

- Pod 的滚动更新：将集群中正在运行的多个 Pod 版本，交替逐一升级的过程。

  ```sh
  # 触发
  $ kubectl edit deployment/xxx
  $ kubectl set image deployment/xxx nginx=ningx:1.20
  
  # 回滚
  $ kubectl rollout undo deployment/xx
  
  ```

  

**原理：**

- 实际上是一个”两层控制器“，依赖 **ReplicaSet** 这个 API 对象

  - Deployment 控制 ReplicaSet
  - ReplicaSet 控制 Pod

- 通过 **ReplicaSet 的个数** 来描述应用的版本；通过 **ReplicaSet 的属性** 来保证Pod的副本数量。

  ```sh
  # edit deployment 后会触发滚动更新
  1. 创建一个新的 ReplicaSet，初始 Pod 副本数是 0 
  2. 将新 ReplicaSet Pod 副本数增加一个
  3. 将旧 ReplicaSet Pod 副本数减少一个
  ```



Tips:

每次修改 deployment 都会生成 新 ReplicaSet 对象，浪费资源。

解决：kubectl rollout pause

```sh
$ kubectl rollout pause deployment/xxx
$ kubectl edit / set image
$ kubectl ...
$ kubectl rollout resume deployment/xxx #此时才会创建一个ReplicaSet
```





**Deployment Yaml 字段**

- RollingUpdateStrategy

  ```yaml
  spec:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1 # 在一次滚动中，可以创建多少个新 Pod
        maxUnavailable: 1 # 在一次滚动中，可以删除多少个旧 Pod
  ```

  

## || StatefulSet

**目的**

- Deployment （ReplicaSet） 解决“无状态应用”，StatefulSet 解决“有状态应用”。



**原理**

记录应用状态，在Pod被重新创建时能够为新Pod恢复这些状态。

- **拓扑状态**：例如主从、按顺序启动 

  > 编号：将 Pod 的拓扑状态，按照 Pod "名字 + 编号"的方式固定下来。

  - 在创建 Pod 的过程中，StatefulSet 给每个Pod的名字进行编号：`StatefulSetName-编号` 。严格按照编号顺序创建。“编号0” 进入 Running 状态之前，“编号1” 一直处于 Pending 状态。
  - 为每个 Pod 创建了唯一且不可变的“网络身份” （`StatefulSetName-编号.ServiceName`），保证 Pod 网络标识的稳定性。
  - 通过 <u>Headless Service</u> 为每个 Pod 生成带有同样编号的 DNS 记录。

  

- **存储状态**：例如一个数据库应用的多个存储实例。

  > PVC：重启 Pod 后会根据特定的名称查到旧 Pod 遗留下来的同名 PVC

  - StatefulSet 为每个 Pod 分配并创建一个"同样编号"的 PVC。
  - 即使 Pod 被删除，所对应的 PVC / PV 依然会被保留下来。



**滚动更新：**

- 只要修改了StatefulSet 的 Pod 模板，就会自动触发滚动更新。

  ```sh
  $ kubectl patch statefulset mysql --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"mysql:5.7.23"}]'
  ```



**版本控制**

Q: 不像deployment 每次修改都会生成 新 ReplicaSet 对象，StatefulSet 直接管理 Pod，那么他是如何管理版本？

A: **ControllerRevision**

- ControllerRevision 专门用来记录控制器对象（StatefulSet, DaemonSet）的版本。
- 原理：在data字段保存完整的API对象，在annotation字段保存创建该对象所使用的的 kubectl 命令。



**Yaml**：类似 Deployment，多了 `spec.serviceName`

```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None  #Headless Serivce，以便暴露 Pod DNS
  selector:
    app: nginx
    
# StatefulSet    
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx" #比Deployment多的字段：Headless Service Name 
  replicas: 2
  selector:  #该StatefulSet要管理的pod必须携带特定label
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts: # PVC
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates: # PVC
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce  #可读写，且一个PV只能挂载在一个宿主机上
      resources:
        requests:
          storage: 1Gi
```



实例：

- MySQL 集群：https://github.com/oracle/kubernetes-website/blob/master/docs/tasks/run-application/mysql-statefulset.yaml
- 



## || Service

**作用：**

- Pod 的 IP 不固定；
- 负载均衡；



**Yaml**

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  #selector 声明这个 Service 只代理携带了 app=hostnames 标签的 Pod
  selector: 
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80 #这个 Service 的 80 端口，代理的是 Pod 的 9376 端口。
    targetPort: 9376
```



**spec.type**

目的：为了从 k8s 集群外部，访问到 k8s 里创建的 Service. 

> Service 的访问信息在 Kubernetes 集群之外，其实是无效的。



- **NodePort**

  ```yaml
# NodePort Service
  apiVersion: v1
  kind: Service
  metadata:
    name: my-nginx
    labels:
      run: my-nginx
  spec:
    type: NodePort  #spec.type
    ports:
    - nodePort: 8080 #暴露端口
      targetPort: 80 #Pod端口
      protocol: TCP
      name: http
    - nodePort: 443
      protocol: TCP
      name: https
    selector:
      run: my-nginx
  ```
  
  - 访问方式：`<任何一台宿主机IP>:8080`

  - IP 包离开宿主机发往 目的Pod时，会做一次 SNAT 操作。

- **LoadBalancer**

  ```yaml
  # LoadBalancer Service
  apiVersion: v1
  kind: Service
  metadata:
    name: example-service
  spec:
    type: LoadBalancer #spec.type
    ports:
    - port: 8765
      targetPort: 9376
    selector:
      app: example
    
  ```
  
  - 适用于公有云上的 k8s 服务：k8s 在 LB Service 被创建后，会调用 CloudProvider 在公有云上创建一个负载均衡服务，并把被代理的 Pod IP 配置为负载均衡器后端。
  
- **ExternalName**

  ```yaml
  # ExternalName Service
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
  spec:
    type: ExternalName #spec.type
    externalName: my.database.example.com
  ```
  
  - 通过 Service 的 DNS 名字访问它的时候，返回的即是 externalName。



**Endpoints**

Service Endpoints 是指被 selector 选中的 Pod；可以通过 `kubectl get ep` 命令查看。

```sh
$ kubectl get endpoints <svc-name>
```



**访问方式**

- **VIP**

  - Service 的虚拟 IP 

    > ClusterIP 模式的 Service ?

- **DNS**

  - 细分1：**Normal Service**，`svc-name.namespace-name.svc.cluster.local`，解析后得到 Service VIP
  - 细分2：**Headless Service**， `pod-name.svc-name.namespace-name.svc.cluster.local`，解析后得到某个 Pod 的 IP 地址



**原理**

kube-proxy 组件 + iptables

- 创建 Service 时，Kube-proxy组件在宿主机上创建一条 iptables 规则；
- Iptables 规则最终指向 pod.



Q：当宿主机有大量 Pod，就会有大量 iptables 规则，会大量占用宿主机CPU资源。

A：IPVS 模式的 Service.





## || Ingress

https://time.geekbang.org/column/article/69214

作用

- **是 Service 的 Service**；是k8s 对 ”**反向代理**“ 的抽象；通过访问的 URL，转发给不同的 Serivce；
- 避免每个 Service 都要一个 负载均衡服务，降低成本。



yaml

```yaml
# ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules: # IngressRule
  - host: cafe.example.com
    http:
      paths:
      - path: /tea    #根据不同路径，转发到不同的 Service
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```



Ingress Controller

- Ingress 相当于接口，具体的 Ingress Controller 根据 Ingress 定义 提供对应的代理能力。

- 种类：Nginx, HAProxy, Envoy, Traefik

- 部署：

  ```sh
  $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
  ```

  



Example: Nginx Ingress Controller

- https://github.com/resouer/kubernetes-ingress/tree/master/examples/complete-example



## || Volume

### emptyDir & hostPath

https://kubernetes.io/docs/concepts/storage/volumes/#emptydir

https://www.infoq.cn/article/ah1n57f8tge2wisquj00

`emptyDir`

- emptyDir 类型的 Volume 在 Pod 分配到 Node 上时被创建，Kubernetes 会在 Node 上自动分配一个目录，因此无需指定宿主机 Node 上对应的目录文件。 
- 这个目录的初始内容为空，当 Pod 从 Node 上移除时，emptyDir 中的数据会被永久删除。

`hostPath`

- hostPath 类型则是映射 node 文件系统中的文件或者目录到 pod 里。
- 即便 pod 已经被删除了，volume 卷中的数据还在。



### PVC & PV

目的：隐藏 Volume 的管理、存储服务器的地址等敏感信息。

- PVC: Persistent Volume Claim，相当于接口；屏蔽存储细节。
- PV: Persistent Volume，相当于实现，通常由运维提供。

> **持久化 Volume**：宿主机上的目录，具备“持久性”。即：这个目录里面的内容，既不会因为容器的删除而被清理掉，也不会跟当前的宿主机绑定。这样，当容器被重启或者在其他节点上重建出来之后，它仍然能够通过挂载这个 Volume，访问到这些内容。
>
> hostPath, emptyDir 则不属于“持久化 Volume”。



PVC 定义

```yaml
# PVC
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  storageClassName: manual # 用于自动创建PV
  accessModes:
  - ReadWriteOnce # 可读写，且一个PV只能挂载在一个宿主机上
  resources:
    requests:
      storage: 1Gi
       
```



运维定义 PV

```yaml
# PV
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
    # 使用 kubectl get pods -n rook-ceph 查看 rook-ceph-mon- 开头的 POD IP 即可得下面的列表
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
```



**Q：PVC 和 PV 是如何关联上的？**

A: PersistentVolumeController，检查条件：

- spec 字段要匹配，例如 PV 的 storage 大小要满足 PVC 的要求；

- storageClassName 要一致；



**Volume Controller**

Volume Controller 维护着多个控制循环，其中有一个循环是 `PersistentVolumeController`，用来帮 PVC 找到合适的 PV，进行绑定。

- 绑定：将PV名称填到 PVC - spec.volumeName字段。



**StorageClass**

> **Q: 如何自动创建 PV？**否则运维需要一个个手工创建 PV
>
> A: Dynamic Provisioning，通过 `StorageClass` 对象作为创建 PV 的模板。运维人员只需创建有限个 StorageClass对象即可，<u>当开发人员提交了包含StorageClass字段的PVC后，k8s会据此创建出对应的PV</u>。



StorageClass 相当于 PV 的模板， 定义如下内容：

- PV 的属性，例如存储类型、Volume大小；
- 需要用到的存储插件，例如Ceph等；

```yaml
# StorageClass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd #指定PV的存储插件！！
parameters: #属性
  type: pd-ssd
```

> PVC 里需要引用 storageClassName



Pod 里使用这个PVC

```yaml
# POD
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes: # 使用PVC
    - name: pv-storage
      persistentVolumeClaim: 
        claimName: pv-claim
```



**Local Persistent Volume**

Local PV 定义

```yaml
#local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  storageClassName: local-storage
  local: #LOCAL
    path: /mnt/disks/vol1
  nodeAffinity: #如果Pod要使用此PV，则必须运行在指定node
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
```

Local StorageClass

```yaml
#local-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer #延迟绑定，否则affinity可能冲突
```





### Projected Volume

Projected Volume 是一种特殊的 Volume，它不是为了存放容器里的数据，而是为容器提供预先定义好的数据。

仿佛是被 k8s 投射进容器当中的。



### Secret

把 Pod 想要访问的加密数据，存放到 **Etcd** 中。然后就可以通过在 Pod 的容器里**挂载 Volume** 的方式，访问到这些 加密数据了。

kubectl create

```sh
$ kubectl create secret generic user --from-file=./username.txt

$ kubectl get secrets
```

或者用 yaml 创建

```yaml
apiVersion: v1
kind: Secret
metadata: 
  name: mysecret
type: Opaque
data: 
  user: YWRtaW4= //base64 encoded
  pass: xxxx
```

> Base64 encode/decode:
>
> ​	echo -n 'password' | base64
>
> ​	echo 'MWYyZDFlMmU2N2Rm' | base64 --decode 



**在 Pod 中使用 Secret**

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume 
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

- /projected-volume 目录下，会有两个文件 user, pass
- Etcd 数据更新后，文件内容也会同步更新 - 有延时



#### ConfigMap

与 Secrect 创建几乎一样。

```sh
# 从.properties文件创建ConfigMap
$ kubectl create configmap ui-config --from-file=example/ui.properties

# 查看这个ConfigMap里保存的信息(data)
$ kubectl get configmaps ui-config -o yaml

apiVersion: v1
data:
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  name: ui-config
  ...
```



#### Downward API

作用：让 Pod 里的容器能否直接获取到这个 Pod API 对象本身的信息。

```

apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    ...
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

- /etc/podinfo/labels 文件中会有当前 Pod 的 Labels 字段值。



#### ServiceAccountToken

是一种特殊的 Secret，存储 Service Account 的授权信息。

- 每个 Pod 都被自动声明 default-token-xxxx 的 Volume；
- 路径：/var/run/secrets/kubernetes.io/serviceaccount



## || Batch

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never #离线计算的Pod永远不该被重启
  backoffLimit: 4 #离线作业失败后Job Controller会不断尝试创建新Pod，直至backoffLimit
  activeDeadlineSeconds: 100 #最长运行时间
  
  parallelism: 2 #并行度，最多可以启动多少个Pod同时运行
  completions: 2 #Job的最小完成数
```



控制器：Job Controller

- Job Controller 控制的对象，直接就是 Pod；
- Job Controller 控制循环：Reconcile，根据 running、completed Pod 数目 & parallelism、completions 计算出在这个周期里应该创建或删除的Pod数目。



使用 Job 的模式：

- 外部管理器 + Job 模板

  yaml里定义变量，创建job时替换变量（例如用 shell sed）；

- 拥有固定任务数目的并行 Job

  指定 completions；

- 执行 parallelism，但不设置固定的 completions

  需要在Pod逻辑里判断何时任务完成、退出；



### CronJob

是一个 Job 对象的控制器！

```yaml

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Allow
  jobTemplate: 
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

concurrencyPolicy: Job还没执行完，新Job又产生了

- Allow：允许同时存在多个Job；
- Forbid：该创建周期被跳过；
- Replace：没执行完的Job会忽略；





## || 工具对象

### PodPreset

作用：自动给对应的 Pod 加上其他必要的信息，例如 labels, annotations, volumes等。

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    # spec.selector 用于选择pod
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```



运用过 PodPreset 的 Pod 会被自动加上一个annotation:

`metadata.annotations: podpreset.admission.kubernetes.io/podpreset-allow-database: "resource version"`

> 如果针对同一Pod定义了多个 PodPreset，则会自动合并，冲突字段不会被修改。



### DaemonSet

作用：在k8s集群里运行 Daemon Pod

- Daemon Pod 运行在k8s集群里的每个节点上；
- 每个节点只有一个这样的 Pod 实例；
- 当新节点加入集群，该 Pod 就会自动在该新节点创建出来。



原理：控制器模型

- 在控制循环中，从 Etcd 获取所有节点列表，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。



yaml：

- 类似 Deployment，但没有 replicas 字段。



设置

- 限制某些节点生效：`nodeAffinity`

  ```yaml
  # pod nodeAffinity
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: metadata.name
              operator: In
              values:
              - node-geektime
  ```

  

- 在某些污点节点(Taint)也生效：`toleration` 

  - 以此实现在 NotReady （e.g. NetworkReady = false）的节点上运行 Daemon Pod。

  ```yaml
  # pod tolerations
  spec:
    tolerations:
    - key: node.kubernetes.io/unschedulable
      operator: Exists
      effect: NoSchedule
  ```

  



# | 编排



## || 控制器模型

**场景：**

例如通过 Deployment 管理 Pod。



**编排模式：control loop**

- 又称`调谐循环`（`Reconcile Loop`），或者`同步循环`（`Sync Loop`）
- 调谐结果：创建、更新一些 Pod（或者其他的 API 对象、资源），要么就是删除一些已经存在的 Pod（或者其他的 API 对象、资源）。

```java
for {
  实际状态 := 获取集群中对象X的实际状态（Actual State，来自集群本身）
  期望状态 := 获取集群中对象X的期望状态（Desired State，来自yaml）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    // Reconcile调谐
    执行编排动作，将实际状态调整为期望状态
  }
}
```



## || 声明式API

- kubectl apply: 对原有 API对象的 Patch 操作；	
  - 一次能处理多个写操作，并具备 Merge 能力。
- kubectl replace: 替换原有的 API 对象；



## || Dynamic Admission Control

Admission:

- 当一个API对象被提交给 API Server之后，在正式处理之前需要进行一些”初始化“性质的工作，例如自动加上某些 Label。
- Admission Controller 可以选择性地编译进 API Server 中，在API 对象创建之后被立即调用。



Dynamic Admission Control (**Initializer**)

- 热插拔式的 Admission机制，无需重新编译重启 API Server；
- https://github.com/resouer/kubernetes-initializer-tutorial



步骤（例： lstio 为 pod 自动注入 Envoy 容器）

- 定义 Envoy 容器，保存到 ConfigMap

  ```yaml
  
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: envoy-initializer
  data:
    config: |
      containers:
        - name: envoy
          image: lyft/envoy:845747db88f102c0fd262ab234308e9e22f693a1
          command: ["/usr/local/bin/envoy"]
          args:
            - "--concurrency 4"
            - "--config-path /etc/envoy/envoy.json"
            - "--mode serve"
          ports:
            - containerPort: 80
              protocol: TCP
          resources:
            limits:
              cpu: "1000m"
              memory: "512Mi"
            requests:
              cpu: "100m"
              memory: "64Mi"
          volumeMounts:
            - name: envoy-conf
              mountPath: /etc/envoy
      volumes:
        - name: envoy-conf
          configMap:
            name: envoy
  ```

- 编写 Initializer，作为一个 Pod 部署到 k8s

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      app: envoy-initializer
    name: envoy-initializer
  spec:
    containers:
      - name: envoy-initializer
        image: envoy-initializer:0.0.1
        imagePullPolicy: Always
  ```

  

- CreateTwoWayMergePatch: 合并两个 pod

  - 用户自定义的pod
  - Configmap 里定义的 envoy 容器



## || RBAC

基本概念

- Role：角色。实际是一组规则，定义了一组对 K8S API对象的操作权限； 
- Subject：被作用者。
- RoleBinding：定义角色和被作用者的绑定关系



### Role

Role 是一个 API 对象。

```yaml
# role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace  #指定能产生作用的 namespace
  name: example-role
rules:    # 定义权限规则
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"] # 
```

- 所有权限：verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]



### RoleBinding

RoleBinding 也是一个 API 对象。

```yaml
# role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace #指定能产生作用的 namespace
subjects: #关联Subject
- kind: User # Subject-1: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
  
- kind: ServiceAccount  # Subject-2：ServiceAccount
  name: example-sa 
  namespace: mynamespace
  
roleRef:  #关联Role
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

- `subjects.kind`

  - `User`

    - 通过外部认证服务来提供，例如Keystone.

  - `ServiceAccount`

    ```yaml
    # svc-account.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      namespace: mynamespace
      name: example-sa
    ```

  - `Group`

    - SA 对应的 Group：`system:serviceaccount:<Namespace名字>:<ServiceAccount名字>`

      

依次创建 svc-account, role, role-binding，K8S 会为该 ServiceAccount 分配一个 Secret。

- 这个 Secret 就是用来跟 API Server进行交互的授权文件（Token）；

```sh

$ kubectl get ServiceAccount -n mynamespace -o yaml

- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: 2018-09-08T12:59:17Z
    name: example-sa
    namespace: mynamespace
    resourceVersion: "409327"
    ...
  secrets:
  - name: example-sa-token-vmfg6 #分配了一个 Secret
```



### Pod 引用 ServiceAccount

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: mynamespace
  name: sa-token-test
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
  serviceAccountName: example-sa # 引用ServiceAccount，否则默认用 `default` SA
```

```sh
$ ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt namespace token
```



该 POD 使用 ca.crt 来访问 API Server，且只有 Role 里定义的权限。



### ClusterRole / ClusterRoleBinding

Role / RoleBinding 的问题：

- 必须指定 Namespace，无法作用于 Non-Namespaced 对象（例如 Node）；
- 无法作用于所有 Namespace；



kubectl get clusterroles

- `system:xx` : 给 k8s 系统组件对应的 ServiceAccount 使用。
- `cluster-admin`: 
- `admin`
- `edit`
- `view`



## || 资源管理和调度

### 资源模型&QoS

```yaml
# 容器 yaml 字段
spec.containers[].resources.requests.cpu: "250m"
spec.containers[].resources.requests.memory: "64Mi"
spec.containers[].resources.limits.cpu: "500m"
spec.containers[].resources.limits.memory: "128Mi"
```

- 在调度的时候，kube-scheduler 只会按照 `requests` 的值进行计算。而在真正设置 Cgroups 限制的时候，kubelet 则会按照 `limits` 的值来进行设置。

- 思路：用户在提交 Pod 时，可以声明一个相对较小的 `requests` 值供调度器使用；而 k8s 真正设置给容器 CGroups 的，则是相对较大的 `limits` 值。

  > 在作业被提交后，会主动减小它的资源限额配置，以便容纳更多的作业、提升资源利用率。而当作业资源使用量增加到一定阈值时，会通过“快速恢复”过程，还原作业原始的资源限额，防止出现异常情况。
  >
  > 因为在实际场景中，大多数作业使用到的资源其实远小于它所请求的资源限额。



**QoS 模型**

不同的 requests + limits 设置会将 pod 划分到不同的 QoS 级别当中。

- **Guaranteed**: Pod 内所有容器都设置了 requests 和 limits，并且 requests == limits。
  - `requests.cpu == limits.cpu` 时相当于 cpuset： 将容器绑定到某个CPU的核上。<u>生产环境的在线应用 Pod 推荐使用！</u>
  - DaemonSet Pod 推荐使用 Guaranteed，因为被回收后会立即重建，回收毫无意义。
- **Burstable**: Pod内至少一个容器设置了 requests。
- **BestEffort**: requests和limits都没有设置。



**作用：**当宿主机资源紧张时，用于对 Pod 进行资源回收（Eviction）：`BestEffort --> Bustable --> Guaranteed`



当宿主机的 Eviction 阈值达到后，就会进入 `MemoryPressure` 或者 `DiskPressure` 状态，从而避免新的 Pod 被调度到这台宿主机上。

> K8S 会为这种宿主机打上污点 taint。



默认资源回收阈值：

```
memory.available<100Mi
nodefs.available<10%
nodefs.inodesFree<5%
imagefs.available<15%
```



### 调度器 kube-scheduler

职责：为新创建的 Pod 寻找一个最合适的节点。

原理：两个独立的控制循环

- **Informer Path**

  - **监听**：启动一系列 Informer，监听 Etcd 中 Pod、Node、Service 等 API 对象的变化；
  - **调度队列**：当一个待调度 Pod (nodeName为空) 被创建后，通过 Pod Informer 的 Handler，将其添加到 调度队列。
  - **调度器缓存**：还负责对 scheduler cache 进行更新。

- **Scheduling Path**

  - **过滤**：从调度队列里取出 Pod，并从调度器缓存中取出节点列表，调用 <u>Predicates 算法</u>进行过滤。

    > GeneralPredicates
    >
    > - PodFitsResources：检查 requests 字段 - CPU、内存；
    > - PodFitsHost：检查宿主机名字是否与spec.nodeName 一致；
    > - PodFitsHostPorts：端口是否已占用；
    >
    > - PodMatchNodeSelector：检查 pod nodeSelector / nodeAffinity 是否与节点匹配
    >
    > Volume 相关
    >
    > - NoDiskConflict：多个 Pod 声明挂载的持久化 Volume 是否有冲突；
    > - MaxPDVolumeCountPredicate ：一个节点上某种类型的持久化 Volume 是不是已经超过了一定数目；
    > - VolumeBindingPredicate：Pod 对应 PV 的 nodeAffinity
    >
    > 宿主机相关
    >
    > - PodToleratesNodeTaints：污点Taint vs. Toleration.
    > - NodeMemoryPressurePredicate：检查节点内存是否充足。
    >
    > Pod 相关
    >
    > - 

    

  - **打分**：调用 <u>Priorities 算法</u>对节点打分。

  - **Bind**: 设置 Pod 的 nodeName 字段。

    > Assume 机制：只更新调度器缓存里的 Pod 和 Node 信息，而不远程访问 API Server。
    > 同时创建一个 Goroutine 异步地向 API Server 发起更新 Pod 的请求。

- **Admit**：当 Pod 完成调度需要在某个节点上运行之前，该节点的 kubelet 会进行 Admit 操作 再次验证 GeneralPredicates。（例如资源是否可用、端口是否冲突）



**优先级（Priority）和抢占（Preemption）机制**

目的：调度失败时如何处理？

- 默认会暂时搁置，直到 Pod 被更新，或者集群状态发生变化。
- 但有时希望 挤走 某些低优先级 Pod。



如何定义优先级？

- spec.priorityClassName



抢占机制

- 高优先级 Pod 调度失败时，会视图找一个节点（使得其中低优先级pod被删除后，可以调度过来）。

- 然后将抢占着 的 spec.nominatedNodeName 设置为该节点，并放入 `unschedulableQ`，进入下一个调度周期。

  > 并不会马上抢占，因为优雅退出期间 集群的可调度性可能会发生变化。

  - 调度队列有两个：`activeQ` `unscheduableQ`



# | 监控

## || Prometheus

运行机制

- Pull 抓取 Metrics；
- 保存到 TSDB，例如 OpenTSDB，InfluxDB；

组件

- `Pushgateway`：允许被监控对象以 Push 方式推送 Metrics 数据；
- `Alertmanager`：根据 Metrics 灵活设置报警；
- `Grafana`：可视化界面；



Pull vs. Push

- pull是拉动作，监听者主动调用被监听者的接口；
  push是推动作，被监听者主动上报，监听者被动采集。
- 拉动作有助于监听者自己控制频率和采样量，缺点是需要掌握所有被监听者的地址和端口，也就是要有注册中心；
- 推动作有利于被监听者自己控制上报数量和频率，但有可能对监听者构成额外的压力，同时有信息丢失的风险；



## || Metrics

**数据来源**

宿主机监控数据

- 通过 Node Exporter 暴露；一般以 DaemonSet 方式运行在宿主机上。

- 暴露Load、CPU等信息，see https://github.com/prometheus/node_exporter#enabled-by-default 



K8S 组件 Metrics （`/metrics` API）

- API Server 组件：各个 Controller 的工作队列长度、请求的 QPS、延迟；

- Kubelet 组件：



K8S 核心监控数据 Core Metrics

- 容器 Metrics
  - 每个容器的 CPU、文件系统、内存、网络资源使用情况；
  - 来自于 kubelet 内置的 `cAdvisor` 服务，该服务随 kubelet 启动。

- Pod、Node、Service 等指标



**Metrics Server**

- 作用：把监控数据以标准 K8S API 暴露出来；

  ```
  http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/namespaces/<namespace-name>/pods/<pod-name>
  ```

- 原理

  - 从 kubelet Summary API 采集而来，包含 cAdvisor 数据 + kubelet 本身汇总的信息。
  - Aggregator 模式：/apis/metrics.k8s.io/v1beta1 访问到 `kube-aggregator` 代理，被分发到 `kube-apiserver` 或者 `Metrics Server`。



**USE 原则**

按照如下三个维度来规划资源监控指标：

- 利用率（Utilization），资源被有效利用起来提供服务的平均时间占比；
- 饱和度（Saturation），资源拥挤的程度，比如工作队列的长度；
- 错误率（Errors），错误的数量。

> 关注 ”资源“ 



**RED 原则**

- 每秒请求数量（Rate）；
- 每秒错误数量（Errors）；
- 服务响应时间（Duration）。

> 关注 “服务”



## || 日志

三种日志方案：

1. **在 Node 上部署 logging agent（DaemonSet，例如 Fluentd），将日志文件转发到后端存储里保存起来。**

- 优点：一个节点只需要部署一个 agent，对应用和 Pod 没有侵入性。
- 缺点：要求应用输出的日志，都必须是直接输出到容器的 stdout / stderr 里。



2. **通过一个 sidecar 容器把日志文件重新输出到 sidecar 的 stdout / stderr 上**，再用第一种方案。

   ```yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: counter
   spec:
     containers:
     - name: count
       image: busybox # 输出日志到 /var/log/1.log
       volumeMounts:
       - name: varlog
         mountPath: /var/log
     # sidecar: 输出到stdout    
     - name: count-log-1
       image: busybox
       args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
       volumeMounts:
       - name: varlog
         mountPath: /var/log
     # app,sidecar共享volume
     volumes:
     - name: varlog
       emptyDir: {}
   ```

- 缺点：同一日志文件会有两份。不推荐。



3. **通过一个 sidecar 容器，把日志文件发送到远程存储。**

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox # 输出日志到 /var/log
    volumeMounts:
    - name: varlog
      mountPath: /var/log
      
  - name: count-agent
    image: k8s.gcr.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /etc/fluentd-config/fluentd.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /etc/fluentd-config
      
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap: # Fluentd 的输入源在 ConfigMap 里配置
      name: fluentd-config
```



```yaml
# fluent-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluentd.conf: |
    <source>
      type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>
    
    <source>
      type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>
    
    <match **>
      type google_cloud
    </match>
```



- 优点：部署简单，对宿主机友好；

- 缺点：sidecar消耗资源；kubectl logs 看不到日志输出；

  > 而方案2 的sidecar是用共用 volume，资源消耗不高



Logrotate

- https://linux.die.net/man/8/logrotate



# | 扩展

## || 自定义

### 自定义 API 对象

https://time.geekbang.org/column/article/41876



### 自定义控制器

https://time.geekbang.org/column/article/42076



### Operator

https://time.geekbang.org/column/article/42493

也是一个自定义控制器。























































