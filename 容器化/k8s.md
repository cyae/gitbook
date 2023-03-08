## **架构与组件**  
    *      * Master----控制节点   
* API Server，由API组件构成，负责提供API服务  
      * Scheduler，负责集群调度  
      * Controller Manager  
      * Node----计算节点，本质是通过各种协议对简单容器(如Docker)进行封装   
* kubelet，负责调用容器引擎，无论哪种引擎，都需遵循CRI接口规范  
        * 容器引擎通过OCI规范调用Linux核心3种命令进行隔离/约束  
        * Device Plugin，负责管理宿主机上的硬件设备  
        * Networking/Volume Plugin，为kubelet提供网络通信/持久化服务  
        * 计算节点间通信需要序列化/反序列化，如可用protobuf/json  
    * * **Pod** ：豌豆荚，里面的容器充当一粒粒豌豆，k8s的基本调度单元是Pod   
* 除了 **InitContainer，Pod内的容器地位平等。而Pod之间可以有拓扑关系，如StatefulSet和Operator**  
        * 同一个Pod内的容器共享Net/volume-namespace资源，可高效通信  
      * 根据不同服务的特性，将Pod分为CronJob【定时】、Job【一次性】、StatefulSet【有状态服务】、DaemonSet【单例守护服务】  
      * 什么情况下将容器放入pod？容器互相之间发生直接的文件交换、使用localhost或者Socket文件进行本地通信、会发生非常频繁的远程调用、需要共享某些Linux Namespace  
      * pod设计模式：由于pod内所有容器共享namespace，且地位公平(针对无状态)，因此在创建pod时，先创建虚拟容器infra（即k8s.gcr.io/pause镜像），其他所有功能容器都join此虚拟容器  
      * **Service** ：Pod的ip是不固定的，因此使用Service组件进行反向代理，对外暴露固定地址   
* Service还可以起到负载均衡作用  
      * **Deployment** ：为了高可用，通常需要启动多个相同Pod副本。Deployment通过ReplicaSet间接控制Pod副本  
      * **Secret** ：对于需要认证的场景，将kv键值对存入etcd。在Pod启动时，将etcd中的键值对挂载到指定Volume，完成验证  
      * **Horizontal Pod Autoscaler** ：自动水平拓展  
      * **ConfigMap** ：保存Pod的配置信息到etcd中，用于Node加入Master节点  
    * 一切皆容器思想：三大Master组件，以及etcd也都会被封装成Pod并启动，其对应的yaml配置一般在/etc/kubernetes/manifests路径下，可按需自定义  

## **API** **对象编写**  
    * kind：对象类型  
    * metadata: 存放的是这个对象的元数据   
* name：在每个pod的namespace内唯一标识  
      * labels：用于与spec.selector.matchLabels匹配，与本API对象控制的对象匹配  
    * spec   
      * replicas：对象内pod的重复数   
* 即使是单例pod，也要封装为replicas=1的Deployment，因为k8s默认pod与所在node绑定，如果node所在宿主机挂了，则此pod的restart会失败。设置为 **Deployment则能在其他node上恢复pod**  
      * selector：匹配labels，规定了管理范围  
      * template：定义pod的具体结构，pod也递归地由metadata+spec描述   
* containers：pod内包含容器及其具体网络，挂载信息   
* ImagePullPolicy：镜像拉取策略，默认always，常用IfNotPresent  
          * lifecycle：当容器状态发生变化时执行的钩子，类比interceptor/filter  
        * initContainers：比普通containers先执行，常用于sidecar注入，作为虚悬镜像  
        * volumes：pod的挂载信息   
* emptyDir: {}----将权限交由具体containers  
          * projected：为容器提供预先存好的数据   
* secret：存在etcd的敏感信息，由于k8s定时心跳维护secret，一旦secret更改而处于心跳周期间，将发生脏读   
* 所以在发起数据库连接的代码处写好 **重试和超时** 的逻辑，绝对是个好习惯  
              * ConfigMap：服务的配置文件，如.properties  
              * DownwardAPI：类似反射，对容器暴露pod的信息  
      * nodeSelector/nodeAffinity：Pod只能运行在携带了指定标签（Label）的Node上；否则，它将调度失败  
      * nodeName：一旦Pod的这个字段被赋值，k8s就会认为这个Pod已经经过了调度，调度的结果就是赋值的节点名字。在测试或者调试用来骗过调度器  
      * hostAliases：定义了Pod的hosts文件（比如/etc/hosts）里的内容  
      * **updateStrategy** **：更新策略，默认为滚动更新**  
      * **tolerations** **：状态容忍**  

## **版本控制**  
    * 水平拓展/收缩：kubectl scale deployment * --replicas=[new val]   
* 原理：通过controll loop对比集群中pod真实状态和yaml中api对象的期望状态，不同则更改拓展数  
    * 滚动更新/升级：使用kubectl edit deployment/* 或者 kubectl set image deployment/* nginx=nginx:version 编辑pod字段，自动触发滚动更新   
* 原理：生成新的空replicaSet，并交替地收缩旧replicaSet至0，拓展新replicaSet至期望值  
      * * 查看实时更新状态：kubectl rollout status  
      * 若要求对更新过程进行精细控制（如灰度发布），使用updateStrategy.rollingUpdate.partition:[阈值]  
    * 版本回滚/操作撤销：kubectl rollout undo deployment/*   
      * 原理：滚动地将旧replicaSet拓展至原值，新replicaSet收缩至0  
      * 回滚到指定版本：   
* 先查找目标历史版本kubectl rollout history  
        * kubectl rollout history deployment/* --revision=目标版本  

## 有状态服务  
    * 拓扑状态   
* 通过Headless Service的方式，StatefulSet为每个Pod创建了一个固定并且稳定的DNS，来作为它的访问入口   
* Headless Service本质即无clusterIP字段的普通Service组件，它会为所有selector代理的pod创建`<pod-name>.<svc-name>.<namespace>`.svc.cluster.local格式的DNS  
      * StatefulSet在使用Pod模板创建Pod的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作   
* StatefulSet本质即有serviceName字段的Deployment，通过StatefulSet创建pod时，为每个pod指定编号`<statefulset name>-<ordinal index>`  
        * 而当StatefulSet的"控制循环"发现Pod的"实际状态"与"期望状态"不一致，需要新建或者删除Pod进行"调谐"的时候，它会严格按照这些Pod编号的顺序，逐一完成这些操作。  
      * 以上DNS和pod编号在StatefulSet生命周期内不变，不会因pod的启停删改变  
      * 通过Headless Service，定义pod间的依赖关系，从而实现主从、主主、主备、一主多从等结构，mysql主从同步使用XtraBackup  
    * 存储状态   
* 通过StatefulSet的volumeClaimTemplates字段，定义PVC存储属性，与包含敏感信息的PV绑定后，作为Volume挂载到pod  
      * PV和PVC的绑定关系，在StatefulSet生命周期内不变，不会因pod的启停删改变  

## 守护服务 
* DaemonSet里的单例pod运行在所有节点上，且会随着节点的增删而增删，常用作网络/存储插件agent、日志、监控等   
	* 为防止node过多导致DS随之过多，一般要使用resources字段约束资源  
* DaemonSet直接管理Pod对象，通过nodeAffinity和Toleration这两个调度器的功能，保证了每个节点上有且只有一个Pod   
  * **在DaemonSet的控制循环中，只需要从etcd遍历所有node，然后根据节点上是否有被管理Pod的数目情况，来决定是否要创建或者删除一个Pod**  
        * 删除----直接调用 **kubectl delete**  
        * 创建----通过nodeSelector字段绑定node【弃用】/通过nodeAffinity字段绑定具有目标特征的node  
      * Taint/Toleration，类比反射暴力突破   
* 如果某些node恰好处于不可调度状态，则使用t **olerations** **字段让node容忍调度操作，即可保证pod一定会被创建【如果node异常则不断重试】**  
	* k8s不允许用户在master节点上创建pod，如果需要创建，可使用 **tolerations容忍node-role.kubernetes.io/master污点** 

## 离线作业 
* 阅后即焚型服务，如批量计算，其restartPolicy为Never或OnFailure   
  * Never----如果失败（Failure），则不会重启容器，会指数级restartPod，次数=spec.backoffLimit字段，Job存活时间=activeDeadlineSeconds  
  * **OnFailure** **-- --如果失败，则直接重启容器**  
* 并行计算   
	* **spec.parallelism** **-- --最大并行数**  
	* **spec.completions** **-- --所需Job数**  
	* 使用标识模板 **$ITEM + 外部命令批量生成Job**  
		* **kubectl create -f ./jobs**  
* Job的控制器是CronJob，配合 **Unix Cron表达式** 

## 网络模型 
* k8s使用cni0代替Docker的docker0网桥，并且直接管理pod  
* 具体网络模型与Docker的Flannel----VXLAN模式相同，都是通过overlay网络组织跨主通信  
* 由于pod内容器都会join到infra的namespace，所以在k8s启动时，会先用ConfigMap配置infra的网络栈   
	* 默认veth端口是单向的，所以配置时要开启hairpin模式  
* 服务发现   
	* VIP----Service提供给pod稳定的ip  
      * Headless Service----Service提供给pod稳定的DNS  
* 负载均衡   
	* IPVS模式支持的负载均衡算法有：   
		* 轮询  
		* 最小连接数  
		* 目的地址hash  
		* 源地址hash  
		* 最短期望延迟  
		* 无须队列等待  

## 常用命令
* 通过yaml文件创建一个api对象：kubectl create -f *.yaml  
* 获取指定api对象：kubectl get pod/svc/… -l app=\* 
* 查看api对象具体信息：kubectl describe   
  * 其中events字段显示操作日志  
* 更新对象：kubectl replace -f \*.yaml  
* 创建/更新对象：kubectl apply -f \*.yaml  
* 进入pod：kubectl exec -it \* -- /bin/bash  
* 删除pod：kubectl delete -f \*.yaml  
* 与pod文件交互: kubectl cp  
* 查看pod日志: kubectl logs  

## K8S liveness与readiness的区别   
* 两者都是K8s用来检查服务是否正常的探针，但是用途不一致：   
* livenessProbe：存活探针，来确定是否需要重启业务容器，不仅仅在重启的时候会探测，在业务正常运行的时候也会探测。容器创建LIVE_CHECK_DELAY秒后开始探测，默认值是success，间隔10秒探测一次，连续3次失败会置为失败，只要检测到一次成功就算成功，失败后会重启业务容器  
* readinessProbe：就绪探针，来确定容器是否已经就绪可以接受流量，就绪后启动下个pod的升级。容器创建30秒后开始探测，默认是fail，间隔10秒探测一次，要检测到一次成功就算成功（本pod状态置成3/3，开始升级下个pod），连续30次失败就将本业务容器的录用从k8s移除，不再接收流量。  
- 具体原理可以查看[外链](https://www.cnblogs.com/xuxinkun/p/11785521.html)