<ul>
<li><strong>架构与组件</strong>
<ul>
<li>
<ul>
<li>Master&mdash;&mdash;控制节点
<ul>
<li>API Server，由API组件构成，负责提供API服务</li>
<li>Scheduler，负责集群调度</li>
<li>Controller Manager</li>
</ul>
</li>
<li>Node&mdash;&mdash;计算节点，本质是通过各种协议对简单容器(如Docker)进行封装
<ul>
<li>kubelet，负责调用容器引擎，无论哪种引擎，都需遵循CRI接口规范</li>
<li>容器引擎通过OCI规范调用Linux核心3种命令进行隔离/约束</li>
<li>Device Plugin，负责管理宿主机上的硬件设备</li>
<li>Networking/Volume Plugin，为kubelet提供网络通信/持久化服务</li>
<li>计算节点间通信需要序列化/反序列化，如可用protobuf/json</li>
</ul>
</li>
</ul>
</li>
<li>
<ul>
<li><strong>Pod</strong>：豌豆荚，里面的容器充当一粒粒豌豆，k8s的基本调度单元是Pod
<ul>
<li>除了<strong>InitContainer，Pod内的容器地位平等。而Pod之间可以有拓扑关系，如StatefulSet和Operator</strong></li>
<li>同一个Pod内的容器共享Net/volume-namespace资源，可高效通信</li>
<li>根据不同服务的特性，将Pod分为CronJob【定时】、Job【一次性】、StatefulSet【有状态服务】、DaemonSet【单例守护服务】</li>
<li>什么情况下将容器放入pod？容器互相之间发生直接的文件交换、使用localhost或者Socket文件进行本地通信、会发生非常频繁的远程调用、需要共享某些Linux Namespace</li>
<li>pod设计模式：由于pod内所有容器共享namespace，且地位公平(针对无状态)，因此在创建pod时，先创建虚拟容器infra（即k8s.gcr.io/pause镜像），其他所有功能容器都join此虚拟容器</li>
</ul>
</li>
<li><strong>Service</strong>：Pod的ip是不固定的，因此使用Service组件进行反向代理，对外暴露固定地址
<ul>
<li>Service还可以起到负载均衡作用</li>
</ul>
</li>
<li><strong>Deployment</strong>：为了高可用，通常需要启动多个相同Pod副本。Deployment通过ReplicaSet间接控制Pod副本</li>
<li><strong>Secret</strong>：对于需要认证的场景，将kv键值对存入etcd。在Pod启动时，将etcd中的键值对挂载到指定Volume，完成验证</li>
<li><strong>Horizontal Pod Autoscaler</strong>：自动水平拓展</li>
<li><strong>ConfigMap</strong>：保存Pod的配置信息到etcd中，用于Node加入Master节点</li>
</ul>
</li>
<li>一切皆容器思想：三大Master组件，以及etcd也都会被封装成Pod并启动，其对应的yaml配置一般在/etc/kubernetes/manifests路径下，可按需自定义</li>
</ul>
</li>
<li><strong>API</strong><strong>对象编写</strong>
<ul>
<li>kind：对象类型</li>
<li>metadata: 存放的是这个对象的元数据
<ul>
<li>name：在每个pod的namespace内唯一标识</li>
<li>labels：用于与spec.selector.matchLabels匹配，与本API对象控制的对象匹配</li>
</ul>
</li>
<li>spec
<ul>
<li>replicas：对象内pod的重复数
<ul>
<li>即使是单例pod，也要封装为replicas=1的Deployment，因为k8s默认pod与所在node绑定，如果node所在宿主机挂了，则此pod的restart会失败。设置为<strong>Deployment则能在其他node上恢复pod</strong></li>
</ul>
</li>
<li>selector：匹配labels，规定了管理范围</li>
<li>template：定义pod的具体结构，pod也递归地由metadata+spec描述
<ul>
<li>containers：pod内包含容器及其具体网络，挂载信息
<ul>
<li>ImagePullPolicy：镜像拉取策略，默认always，常用IfNotPresent</li>
<li>lifecycle：当容器状态发生变化时执行的钩子，类比interceptor/filter</li>
</ul>
</li>
<li>initContainers：比普通containers先执行，常用于sidecar注入，作为虚悬镜像</li>
<li>volumes：pod的挂载信息
<ul>
<li>emptyDir: {}&mdash;&mdash;将权限交由具体containers</li>
<li>projected：为容器提供预先存好的数据
<ul>
<li>secret：存在etcd的敏感信息，由于k8s定时心跳维护secret，一旦secret更改而处于心跳周期间，将发生脏读
<ul>
<li>所以在发起数据库连接的代码处写好<strong>重试和超时</strong>的逻辑，绝对是个好习惯</li>
<li>ConfigMap：服务的配置文件，如.properties</li>
<li>DownwardAPI：类似反射，对容器暴露pod的信息</li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
<li>nodeSelector/nodeAffinity：Pod只能运行在携带了指定标签（Label）的Node上；否则，它将调度失败</li>
<li>nodeName：一旦Pod的这个字段被赋值，k8s就会认为这个Pod已经经过了调度，调度的结果就是赋值的节点名字。在测试或者调试用来骗过调度器</li>
<li>hostAliases：定义了Pod的hosts文件（比如/etc/hosts）里的内容</li>
<li><strong>updateStrategy</strong><strong>：更新策略，默认为滚动更新</strong></li>
<li><strong>tolerations</strong><strong>：状态容忍</strong></li>
</ul>
</li>
</ul>
</li>
<li><strong>版本控制</strong>
<ul>
<li>水平拓展/收缩：kubectl scale deployment * --replicas=[new val]
<ul>
<li>原理：通过controll loop对比集群中pod真实状态和yaml中api对象的期望状态，不同则更改拓展数</li>
</ul>
</li>
<li>滚动更新/升级：使用kubectl edit deployment/* 或者 kubectl set image deployment/* nginx=nginx:version 编辑pod字段，自动触发滚动更新
<ul>
<li>原理：生成新的空replicaSet，并交替地收缩旧replicaSet至0，拓展新replicaSet至期望值</li>
<li></li>
<li>查看实时更新状态：kubectl rollout status</li>
<li>若要求对更新过程进行精细控制（如灰度发布），使用updateStrategy.rollingUpdate.partition:[阈值]</li>
</ul>
</li>
<li>版本回滚/操作撤销：kubectl rollout undo deployment/*
<ul>
<li>原理：滚动地将旧replicaSet拓展至原值，新replicaSet收缩至0</li>
<li>回滚到指定版本：
<ul>
<li>先查找目标历史版本kubectl rollout history</li>
<li>kubectl rollout history deployment/* --revision=目标版本</li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
<li><strong>有状态服务</strong>
<ul>
<li>拓扑状态
<ul>
<li>通过Headless Service的方式，StatefulSet为每个Pod创建了一个固定并且稳定的DNS，来作为它的访问入口
<ul>
<li>Headless Service本质即无clusterIP字段的普通Service组件，它会为所有selector代理的pod创建&lt;pod-name&gt;.&lt;svc-name&gt;.&lt;namespace&gt;.svc.cluster.local格式的DNS</li>
</ul>
</li>
<li>StatefulSet在使用Pod模板创建Pod的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作
<ul>
<li>StatefulSet本质即有serviceName字段的Deployment，通过StatefulSet创建pod时，为每个pod指定编号&lt;statefulset name&gt;-&lt;ordinal index&gt;</li>
<li>而当StatefulSet的&ldquo;控制循环&rdquo;发现Pod的&ldquo;实际状态&rdquo;与&ldquo;期望状态&rdquo;不一致，需要新建或者删除Pod进行&ldquo;调谐&rdquo;的时候，它会严格按照这些Pod编号的顺序，逐一完成这些操作。</li>
</ul>
</li>
<li>以上DNS和pod编号在StatefulSet生命周期内不变，不会因pod的启停删改变</li>
<li>通过Headless Service，定义pod间的依赖关系，从而实现主从、主主、主备、一主多从等结构，mysql主从同步使用XtraBackup</li>
</ul>
</li>
<li>存储状态
<ul>
<li>通过StatefulSet的volumeClaimTemplates字段，定义PVC存储属性，与包含敏感信息的PV绑定后，作为Volume挂载到pod</li>
<li>PV和PVC的绑定关系，在StatefulSet生命周期内不变，不会因pod的启停删改变</li>
</ul>
</li>
<li></li>
</ul>
</li>
<li><strong>守护服务</strong>
<ul>
<li>DaemonSet里的单例pod运行在所有节点上，且会随着节点的增删而增删，常用作网络/存储插件agent、日志、监控等
<ul>
<li>为防止node过多导致DS随之过多，一般要使用resources字段约束资源</li>
</ul>
</li>
<li>DaemonSet直接管理Pod对象，通过nodeAffinity和Toleration这两个调度器的功能，保证了每个节点上有且只有一个Pod
<ul>
<li><strong>在DaemonSet的控制循环中，只需要从etcd遍历所有node，然后根据节点上是否有被管理Pod的数目情况，来决定是否要创建或者删除一个Pod</strong>
<ul>
<li>删除&mdash;&mdash;直接调用<strong>kubectl delete</strong></li>
<li>创建&mdash;&mdash;通过nodeSelector字段绑定node【弃用】/通过nodeAffinity字段绑定具有目标特征的node</li>
</ul>
</li>
<li>Taint/Toleration，类比反射暴力突破
<ul>
<li>如果某些node恰好处于不可调度状态，则使用t<strong>olerations</strong><strong>字段让node容忍调度操作，即可保证pod一定会被创建【如果node异常则不断重试】</strong></li>
<li>k8s不允许用户在master节点上创建pod，如果需要创建，可使用<strong>tolerations容忍node-role.kubernetes.io/master污点</strong></li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
<li><strong>离线作业</strong>
<ul>
<li>阅后即焚型服务，如批量计算，其restartPolicy为Never或OnFailure
<ul>
<li>Never&mdash;&mdash;如果失败（Failure），则不会重启容器，会指数级restartPod，次数=spec.backoffLimit字段，Job存活时间=activeDeadlineSeconds</li>
<li><strong>OnFailure</strong><strong>&mdash;&mdash;如果失败，则直接重启容器</strong></li>
</ul>
</li>
<li>并行计算
<ul>
<li><strong>spec.parallelism</strong><strong>&mdash;&mdash;最大并行数</strong></li>
<li><strong>spec.completions</strong><strong>&mdash;&mdash;所需Job数</strong></li>
<li>使用标识模板<strong>$ITEM + 外部命令批量生成Job</strong>
<ul>
<li><strong>kubectl create -f ./jobs</strong></li>
</ul>
</li>
</ul>
</li>
<li>Job的控制器是CronJob，配合<strong>Unix Cron表达式</strong></li>
</ul>
</li>
<li><strong>网络模型</strong>
<ul>
<li>k8s使用cni0代替Docker的docker0网桥，并且直接管理pod</li>
<li>具体网络模型与Docker的Flannel&mdash;&mdash;VXLAN模式相同，都是通过overlay网络组织跨主通信</li>
<li>由于pod内容器都会join到infra的namespace，所以在k8s启动时，会先用ConfigMap配置infra的网络栈
<ul>
<li>默认veth端口是单向的，所以配置时要开启hairpin模式</li>
</ul>
</li>
<li>服务发现
<ul>
<li>VIP&mdash;&mdash;Service提供给pod稳定的ip</li>
<li>Headless Service&mdash;&mdash;Service提供给pod稳定的DNS</li>
</ul>
</li>
<li>负载均衡
<ul>
<li>IPVS模式支持的负载均衡算法有：
<ul>
<li>轮询</li>
<li>最小连接数</li>
<li>目的地址hash</li>
<li>源地址hash</li>
<li>最短期望延迟</li>
<li>无须队列等待</li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
<li><strong>常用命令</strong>
<ul>
<li>通过yaml文件创建一个api对象：kubectl create -f *.yaml</li>
<li>获取指定api对象：kubectl get [pod/svc/&hellip;] [-l app=*]</li>
<li>查看api对象具体信息：kubectl describe
<ul>
<li>其中events字段显示操作日志</li>
</ul>
</li>
<li>更新对象：kubectl replace -f *.yaml</li>
<li>创建/更新对象：kubectl apply -f *.yaml</li>
<li>进入pod：kubectl exec -it * -- /bin/bash</li>
<li>删除pod：kubectl delete -f *.yaml</li>
<li>与pod文件交互: kubectl cp</li>
<li>查看pod日志: kubectl logs</li>
</ul>
</li>
<li>K8S liveness与readiness的区别
<ul>
<li>两者都是K8s用来检查服务是否正常的探针，但是用途不一致：
<p>&nbsp;</p>
</li>
<li>
<p>livenessProbe：存活探针，来确定是否需要重启业务容器，不仅仅在重启的时候会探测，在业务正常运行的时候也会探测。容器创建LIVE_CHECK_DELAY秒后开始探测，默认值是success，间隔10秒探测一次，连续3次失败会置为失败，只要检测到一次成功就算成功，失败后会重启业务容器</p>
</li>
<li>
<p>readinessProbe：就绪探针，来确定容器是否已经就绪可以接受流量，就绪后启动下个pod的升级。容器创建30秒后开始探测，默认是fail，间隔10秒探测一次，要检测到一次成功就算成功（本pod状态置成3/3，开始升级下个pod），连续30次失败就将本业务容器的录用从k8s移除，不再接收流量。</p>
<p>具体原理可以查看外链：https://www.cnblogs.com/xuxinkun/p/11785521.html</p>
</li>
</ul>
</li>
</ul>
