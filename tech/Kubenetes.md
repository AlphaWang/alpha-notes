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





## 原理



**Namespace**

- 作用：**隔离**。例如容器内部 ps，只能看到该容器内的进程
- PID Namespace: 
  - 对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号，比如 PID=1。可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。
- Mount Namespace
  - 原理：**chroot**
    例如使用 $HOME/test 目录作为 /bin/bash 进程的根目录：`$ chroot $HOME/test /bin/bash`
- UTS Namespace
- Network Namespace
- User Namespace



**Cgroups**

- Linux Control Group
- 作用：**限制**一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。
- 原理：
  - 以文件形式组织在 /sys/fs/cgroup
  - 控制组：/sys/fs/cgroup/xx
  - 子目录：cpu, blkio, cpuset, memory



**容器镜像**

- 容器镜像：挂载在容器根目录的文件系统，**rootfs**
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













