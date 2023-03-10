---
date created: 2023-03-10 10:44
---

#运维 #容器

## **架构与组件**

- Master----控制节点
  - API Server，由 API 组件构成，负责提供 API 服务
  - Scheduler，负责集群调度
  - Controller Manager
- Node----计算节点，本质是通过各种协议对简单容器(如 Docker)进行封装
  - kubelet，负责调用容器引擎，无论哪种引擎，都需遵循 CRI 接口规范
  - 容器引擎通过 OCI 规范调用 Linux 核心 3 种命令进行隔离/约束
  - Device Plugin，负责管理宿主机上的硬件设备
  - Networking/Volume Plugin，为 kubelet 提供网络通信/持久化服务
  - 计算节点间通信需要序列化/反序列化，如可用 protobuf/json
- **Pod** ：豌豆荚，里面的容器充当一粒粒豌豆，k8s 的基本调度单元是 Pod
  - 除了 **InitContainer，Pod 内的容器地位平等。而 Pod 之间可以有拓扑关系，如 StatefulSet 和 Operator**
  - 同一个 Pod 内的容器共享 Net/volume-namespace 资源，可高效通信
  - 根据不同服务的特性，将 Pod 分为 CronJob【定时】、Job【一次性】、StatefulSet【有状态服务】、DaemonSet【单例守护服务】
  - 什么情况下将容器放入 pod？容器互相之间发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace
  - pod 设计模式：由于 pod 内所有容器共享 namespace，且地位公平(针对无状态)，因此在创建 pod 时，先创建虚拟容器 infra（即 k8s.gcr.io/pause 镜像），其他所有功能容器都 join 此虚拟容器
- **Service** ：Pod 的 ip 是不固定的，因此使用 Service 组件进行反向代理，对外暴露固定地址
  - Service 还可以起到负载均衡作用
- **Deployment** ：为了高可用，通常需要启动多个相同 Pod 副本。Deployment 通过 ReplicaSet 间接控制 Pod 副本
- **Secret** ：对于需要认证的场景，将 kv 键值对存入 etcd。在 Pod 启动时，将 etcd 中的键值对挂载到指定 Volume，完成验证
- **Horizontal Pod Autoscaler** ：自动水平拓展
- **ConfigMap** ：保存 Pod 的配置信息到 etcd 中，用于 Node 加入 Master 节点
- 一切皆容器思想：三大 Master 组件，以及 etcd 也都会被封装成 Pod 并启动，其对应的 yaml 配置一般在/etc/kubernetes/manifests 路径下，可按需自定义

## **API** **对象编写**

- kind：对象类型
- metadata: 存放的是这个对象的元数据
  - name：在每个 pod 的 namespace 内唯一标识
  - labels：用于与 spec.selector.matchLabels 匹配，与本 API 对象控制的对象匹配
- spec
  - replicas：对象内 pod 的重复数
    - 即使是单例 pod，也要封装为 replicas=1 的 Deployment，因为 k8s 默认 pod 与所在 node 绑定，如果 node 所在宿主机挂了，则此 pod 的 restart 会失败。设置为 **Deployment 则能在其他 node 上恢复 pod**
  - selector：匹配 labels，规定了管理范围
  - template：定义 pod 的具体结构，pod 也递归地由 metadata+spec 描述
    - containers：pod 内包含容器及其具体网络，挂载信息
      - ImagePullPolicy：镜像拉取策略，默认 always，常用 IfNotPresent
      - lifecycle：当容器状态发生变化时执行的钩子，类比 interceptor/filter
    - initContainers：比普通 containers 先执行，常用于 sidecar 注入，作为虚悬镜像
    - volumes：pod 的挂载信息
      - emptyDir: {}----将权限交由具体 containers
      - projected：为容器提供预先存好的数据
        - secret：存在 etcd 的敏感信息，由于 k8s 定时心跳维护 secret，一旦 secret 更改而处于心跳周期间，将发生脏读
          - 所以在发起数据库连接的代码处写好 **重试和超时** 的逻辑，绝对是个好习惯
        - ConfigMap：服务的配置文件，如.properties
        - DownwardAPI：类似反射，对容器暴露 pod 的信息
  - nodeSelector/nodeAffinity：Pod 只能运行在携带了指定标签（Label）的 Node 上；否则，它将调度失败
  - nodeName：一旦 Pod 的这个字段被赋值，k8s 就会认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。在测试或者调试用来骗过调度器
  - hostAliases：定义了 Pod 的 hosts 文件（比如/etc/hosts）里的内容
  - **updateStrategy** **：更新策略，默认为滚动更新**
  - **tolerations** **：状态容忍**

## **版本控制**

- 水平拓展/收缩：`kubectl scale deployment * --replicas=new val`
  - 原理：通过 controll loop 对比集群中 pod 真实状态和 yaml 中 api 对象的期望状态，不同则更改拓展数
- 滚动更新/升级：使用`kubectl edit deployment/*` 或者 `kubectl set image deployment/* nginx=nginx:version` 编辑 pod 字段，自动触发滚动更新
  - 原理：生成新的空 replicaSet，并交替地收缩旧 replicaSet 至 0，拓展新 replicaSet 至期望值
    - 查看实时更新状态：`kubectl rollout status`
    - 若要求对更新过程进行精细控制（如灰度发布），使用`updateStrategy.rollingUpdate.partition:阈值`
- 版本回滚/操作撤销：`kubectl rollout undo deployment/*`
  - 原理：滚动地将旧 replicaSet 拓展至原值，新 replicaSet 收缩至 0
  - 回滚到指定版本：
    - 先查找目标历史版本`kubectl rollout history`
    - `kubectl rollout history deployment/* --revision=目标版本`

## 有状态服务

- 拓扑状态
  - 通过 Headless Service 的方式，StatefulSet 为每个 Pod 创建了一个固定并且稳定的 DNS，来作为它的访问入口
    - Headless Service 本质即无 clusterIP 字段的普通 Service 组件，它会为所有 selector 代理的 pod 创建`<pod-name>.<svc-name>.<namespace>.svc.cluster.local`格式的 DNS
  - StatefulSet 在使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作
    - StatefulSet 本质即有 serviceName 字段的 Deployment，通过 StatefulSet 创建 pod 时，为每个 pod 指定编号`<statefulset name>-<ordinal index>`
    - 而当 StatefulSet 的"控制循环"发现 Pod 的"实际状态"与"期望状态"不一致，需要新建或者删除 Pod 进行"调谐"的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。
  - 以上 DNS 和 pod 编号在 StatefulSet 生命周期内不变，不会因 pod 的启停删改变
  - 通过 Headless Service，定义 pod 间的依赖关系，从而实现主从、主主、主备、一主多从等结构，mysql 主从同步使用 XtraBackup
- 存储状态
  - 通过 StatefulSet 的 volumeClaimTemplates 字段，定义 PVC 存储属性，与包含敏感信息的 PV 绑定后，作为 Volume 挂载到 pod
  - PV 和 PVC 的绑定关系，在 StatefulSet 生命周期内不变，不会因 pod 的启停删改变

## 守护服务

- DaemonSet 里的单例 pod 运行在所有节点上，且会随着节点的增删而增删，常用作网络/存储插件 agent、日志、监控等
  - 为防止 node 过多导致 DS 随之过多，一般要使用 resources 字段约束资源
- DaemonSet 直接管理 Pod 对象，通过 nodeAffinity 和 Toleration 这两个调度器的功能，保证了每个节点上有且只有一个 Pod
  - **在 DaemonSet 的控制循环中，只需要从 etcd 遍历所有 node，然后根据节点上是否有被管理 Pod 的数目情况，来决定是否要创建或者删除一个 Pod**
    - 删除----直接调用 **kubectl delete**
    - 创建----通过 nodeSelector 字段绑定 node【弃用】/通过 nodeAffinity 字段绑定具有目标特征的 node
  - Taint/Toleration，类比反射暴力突破
- 如果某些 node 恰好处于不可调度状态，则使用 t **olerations** **字段让 node 容忍调度操作，即可保证 pod 一定会被创建【如果 node 异常则不断重试】**
  - k8s 不允许用户在 master 节点上创建 pod，如果需要创建，可使用 **tolerations 容忍 node-role.kubernetes.io/master 污点**

## 离线作业

- 阅后即焚型服务，如批量计算，其 restartPolicy 为 Never 或 OnFailure
  - Never----如果失败（Failure），则不会重启容器，会指数级 restartPod，次数=spec.backoffLimit 字段，Job 存活时间=activeDeadlineSeconds
  - **OnFailure** **-- --如果失败，则直接重启容器**
- 并行计算
  - **spec.parallelism** **-- --最大并行数**
  - **spec.completions** **-- --所需 Job 数**
  - 使用标识模板 **$ITEM + 外部命令批量生成 Job**
    - **kubectl create -f ./jobs**
- Job 的控制器是 CronJob，配合 **Unix Cron 表达式**

## 网络模型

- k8s 使用 cni0 代替 Docker 的 docker0 网桥，并且直接管理 pod
- 具体网络模型与 Docker 的 Flannel----VXLAN 模式相同，都是通过 overlay 网络组织跨主通信
- 由于 pod 内容器都会 join 到 infra 的 namespace，所以在 k8s 启动时，会先用 ConfigMap 配置 infra 的网络栈
  - 默认 veth 端口是单向的，所以配置时要开启 hairpin 模式
- 服务发现
  - VIP----Service 提供给 pod 稳定的 ip
    - Headless Service----Service 提供给 pod 稳定的 DNS
- 负载均衡
  - IPVS 模式支持的负载均衡算法有：
    - 轮询
    - 最小连接数
    - 目的地址 hash
    - 源地址 hash
    - 最短期望延迟
    - 无须队列等待

## 常用命令

- 通过 yaml 文件创建一个 api 对象：`kubectl create -f *.yaml`
- 获取指定 api 对象：`kubectl get pod/svc/… -l app=*`
- 查看 api 对象具体信息：`kubectl describe`
  - 其中 events 字段显示操作日志
- 更新对象：`kubectl replace -f *.yaml`
- 创建/更新对象：`kubectl apply -f *.yaml`
- 进入 pod：`kubectl exec -it * -- /bin/bash`
- 删除 pod：`kubectl delete -f *.yaml`
- 与 pod 文件交互: `kubectl cp`
- 查看 pod 日志: `kubectl logs`

## K8S liveness 与 readiness 的区别

- 两者都是 K8s 用来检查服务是否正常的探针，但是用途不一致：
- livenessProbe：存活探针，来确定是否需要重启业务容器，不仅仅在重启的时候会探测，在业务正常运行的时候也会探测。容器创建 LIVE_CHECK_DELAY 秒后开始探测，默认值是 success，间隔 10 秒探测一次，连续 3 次失败会置为失败，只要检测到一次成功就算成功，失败后会重启业务容器
- readinessProbe：就绪探针，来确定容器是否已经就绪可以接受流量，就绪后启动下个 pod 的升级。容器创建 30 秒后开始探测，默认是 fail，间隔 10 秒探测一次，要检测到一次成功就算成功（本 pod 状态置成 3/3，开始升级下个 pod），连续 30 次失败就将本业务容器的录用从 k8s 移除，不再接收流量。

- 具体原理可以查看[外链](https://www.cnblogs.com/xuxinkun/p/11785521.html)
