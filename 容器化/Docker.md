---
date created: 2023-03-10 10:43
date updated: 2023-03-10 10:43
---

#运维 #容器

## 特性

- 相比于 VM，Docker 的目标是可装配、轻量化。
- VM 的隔离级别是内核级，但也带来了多余的 overhead，每个 APP 都需要对应一套 OS 和环境，即使有些 APP 对环境的要求是一样的
- Docker 的隔离级别是环境级，通过不同环境隔离不同 APP 组，同一环境上可部署多个 APP（只要他们对环境的要求是一样的），使用 Docker 引擎（daemon）管理这些环境，支撑每个环境的是轻量级 Linux
- Docker 解耦了镜像和容器，镜像是静态的文件，即使运行的容器损坏，也能从镜像快速重启；而对于 VM，二者是高度耦合的，OS 损坏往往需要重装系统，不能快速重启
- 底层实现原理
  - Linux 的 **Namespace** 技术，用于修改当前进程的视图，是一种映射，使在宿主机上的 PID 映射到容器中的 PID，容器中的进程只能看到本容器，看不到其他容器和宿主机进程（如果没暴露的话）
  - Linux 的 **CGroups** 技术，虽然 namespace 做了资源隔离，但由于真实管理者还是 HostOS，可能会使宿主机视角下的容器资源分配出问题，CGroups 可以对容器分配资源
  - 因此，Docker 对容器的管理本质上还是 HostOS 在执行，并没有 VM 中的 Hypervisor 充当管理者
  - 优点：可复用，可装配，轻量级，高性能
  - 缺点：隔离级别较低，安全性没 VM 高，对于无法使用 namespace 的要素（如系统时间）无法做隔离

## 名词解释

- 镜像：一套封装好的环境蓝本，镜像的唯一标识可以是
  - 镜像名：版本号
  - ID
- 容器：运行镜像，这个轻量级的系统启动后就充当容器功能，其内部可盛装 APP
- 仓库：镜像的存储发布管理中心
- 层：镜像是由一系列层堆叠而成的，每一层的功能和所需资源都不同，最大层数为 128。如某镜像 = [Ubuntu + Nginx + JDK + MySQL + APP]，共 5 层。不同镜像指向所需层，相同层是可共享的

## 常用命令

- docker pull xxx 拉取镜像
- docker rmi 删除镜像
- docker rm 删除容器
- docker images 查看本地镜像
- docker run 运行容器
  - -it 进行容器交互
  - --restart=always 容器随引擎启动
  - --rm 容器 stop 后立即删除
  - --add-host 注入解析
- docker ps 列出所有运行中的容器
- docker start 启动容器
- docker restart 重启容器
- docker stop 停止容器
- docker kill 强制停止
- docker build 构建 dockerfile
- docker-compose 规定多个有依赖关系容器的启停顺序
- docker info 查看引擎信息
- docker search 检索容器
- docker stats 查看容器占用资源
- docker inspect 查看容器信息

## 持久化

- 容器一旦被删除，内部数据随之删除，这对数据库等容器很不利，需要将容器中的数据持久化
- 容器与容器间持久化
  - NFS
  - 主机 temp 目录作为文件桥
  - --volumes-from 容器----容器
- 容器与主机间持久化
  - 在 DockerFile 中指定数据卷 VOLUME
  - -v 容器----主机

## 网络通信

- 容器网络模型
  - 同宿主机间容器通信----docker0 虚拟网卡作为端口，转发交由真实网卡 eth0 处理
  - 跨宿主机间容器通信----利用 Overlay 网络进行隧道通信，比如三层的 FlannelUDP 模式
    - 用户的容器都连接在 docker0 网桥上。而网络插件则在宿主机上创建一个特殊的设备（UDP 模式创建的是 TUN 设备，VXLAN 模式创建的则是 VTEP 设备），docker0 与这个设备之间，通过 IP 转发（路由表）进行协作
    - 然后，网络插件通过某种方法，把不同宿主机上的特殊设备连通，从而达到容器跨主机通信的目的
- 通信模式：--net
  - host 模式：使用主机 ip:port
  - container 模式：使用指定容器的命名空间 ip:port
  - none 模式：不指定网络模式，可自定义
  - bridge 模式：即 NAT 组网，同一子网的容器对外暴露统一 ip:port，子网间的容器可通过网桥通信，是默认通信方式
  - overlay 模式：在 Docker 集群的内部节点之间创建虚拟根节点，不同内部容器通过虚拟根节点跨主机访问
- 端口映射：-p
  - 容器 port 绑定指定 ip:port
  - 容器 port 绑定指定 ip 的随机 port
  - 容器 port 绑定主机 ip 的指定 port
  - 主机 port:容器 port
  - docker port 查看容器的端口映射
- 容器通信
  - 容器间通信(默认)
    - 使用网桥 docker0 +命名空间完成交换
  - 容器访问主机
    - SNAT
  - 主机访问容器
    - DNAT

## 格式化命令

- 嵌套：类似 sql 子查询，使用$()列出子查询结果，作为外层命令的执行对象
  - 强制删除所有容器：docker rm -f $(docker ps -a -q)

## 制作 DockerFile

- 在基础 OS 上使用 bash 配置好所需环境后，使用 docker commit 生成有状态的镜像
- 使用 DockerFile 生成 linux 命令，创建无状态的镜像
  1. FROM 基础 image
  2. MAINTAINER 创建者
  3. RUN 执行基础 image 命令，与其他 DockerFile 命令配合使用，多条命令用&&连接
  4. CMD/ENTRYPOINT 设置容器启动时执行的指令，多条指令用&&连接
  5. USER 容器启动用户
  6. EXPOSE 暴露指定端口给主机
  7. ENV 注入环境变量 kv
  8. ADD 从源路径复制文件到容器的指定路径，源路径可以是基础 image 中的绝对或相对路径或 url，如果文件是 tar，会被自动解压
  9. COPY 同 ADD，但不支持 url，且不会解压
  10. WORKDIR 切换目录，与 RUN 配合使用
  11. ONBUILD 设置如果当前镜像被制作为新的镜像，则在新镜像中执行，配合 RUN
  12. VOLUME 挂载主机文件映射
  13. 最后运行 docker build -t name:version DockerFile path 生成镜像

## 容器资源设置

- 内存限制
  - --memory b/k/m/g 设置容器最大硬内存
  - --memory-reservation 设置容器最大软内存
  - --oom-kill-disable 设置内存不足时，容器是否会被杀死
- CPU 限制
  - -c 设置容器占用 CPU 权重，默认 1024
  - --cpus 设置容器能使用的 cpu 个数
  - --cpu-period 设置容器的调度周期
  - --cpu-quota 设置容器在每个调度周期内可使用的 CPU 时间
- 生产环境中，制作完 DockerFile 后需进行内存/CPU 压测
