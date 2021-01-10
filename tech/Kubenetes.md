[toc]



# Docker

## 原理



**Namespace**

- 作用：**隔离**。例如容器内部 ps，只能看到该容器内的进程
- 原理：
  - PID Namespace: 
    对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号，比如 PID=1。可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。
  - Mount Namespace
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



容器镜像