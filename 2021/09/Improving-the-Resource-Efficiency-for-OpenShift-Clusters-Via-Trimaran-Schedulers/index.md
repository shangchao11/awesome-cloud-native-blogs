# 资源管理面临的问题
在OpenShift中，作为开发者，您可以通过请求去设置资源，以规定指定容器需要多少资源(CPU、内存和其他)。“Request”是容器保证得到的内容。容器的总请求之后被程序调度器使用，而且它会决定放置在哪个相对空闲的Pod节点上。
# 实践中设置请求资源是十分困难的
为容器设置请求不是一项简单的任务。在实践中，开发者通常会过度供应容器的资源，以避免供应不足的风险，包括由于内存不足而导致的Pod迁移以及由于资源不足而导致的应用程序性能低下等问题。将请求资源设置得远远高于实际使用量会导致严重的资源过度配置和非常低的资源效率。这些资源是保留的，但实践中没有使用。而将请求资源设置得太低或完全忽略请求的突然峰值也不能解决问题，因为对容器没有QoS（Quality of Service，服务质量）保证。
# 基准测试很麻烦，并不是对所有的工作负载都可行
有人可能提倡对实际负载下的容器的资源使用进行基准测试，以这种方式应该为请求设置什么多大的资源。然而，对于涉及数十个微服务的应用程序，单纯地对这些微服务逐个进行基准测试需要大量的手工工作和资源消耗，这与我们提高资源效率的目的背道而驰。此外，为每个微服务生成实际负载量是一项不可能完成的任务，因为您永远不知道用户调用接口的访问量根据时间段，节日的不同会有多大的变动。
# OverCommitting是有风险的
当Pod申请的请求资源太大时，集群将Pod放置在过度提交的节点上肯定会节省更多资源。然而，到底有多少资源是过度供应的呢?一些应用程序可能有一块双核CPU供应资源，而应用程序可能只需要200毫核。如果不考虑它们的实际使用情况，并且它们都被调度到同一个节点上，并且允许过度使用，那么过度申请资源的应用程序最终将在繁忙时间获得更多的资源。然后，所有的开发者都倾向于为他们的应用程序分配更多的资源。最终，总会有一些应用程序或者微服务由于硬件资源被申请完了而没有办法去申请到它所需的资源，所以性能很差。
# 目标:高效的资源管理
集群管理员通常会抱怨集群的总体资源利用率非常低。然而这种场景下，Kubernetes仍然无法在集群中安排更多的工作负载。但是，他们不愿意调整Pod申请资源的大小，因为这些请求被设置为处理高峰场景时使用。这种基于峰值使用的资源分配导致集群规模不断增长，大多数时候计算资源的利用率极低，以及会产生大量的机器成本。



运行稳定生产服务的集群的主要目标是通过有效地使用所有节点来最小化机器成本。为了实现这个目标，Kubernetes调度器可以了解资源分配和实际资源利用之间的差距。利用间隔的调度器可以帮助更有效地打包pod，而默认调度器则不能。目前有两个主要目标:将节点利用率维持在一定的水平，并平衡节点之间的利用率变化。
# 将节点利用率维持在一定的水平
尽可能地提高资源利用率可能不是所有集群的正确解决方案。由于扩展集群以处理突然的负载峰值总是需要时间，集群管理员希望为突然的负载留出足够的空间，以确保有足够的时间向集群添加更多的节点。



根据之前对实际工作负载的观察，集群管理员发现负载具有一定的季节性，和且周期性。然而，在将新节点添加到集群之前，资源利用率如果总是增加x%。集群管理员希望维护集群，使所有节点的平均利用率在100 - x%左右或以下即可。
# 平衡高峰使用的风险
在某些情况下，调度Pod以维持所有节点的平均利用率也是有风险的，因为不知道不同节点的利用率如何变化
![在这里插入图片描述](https://img-blog.csdnimg.cn/82580366ade74685afe5c1b805ccce53.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5ZWG5pyd,size_20,color_FFFFFF,t_70,g_se,x_16)
例如，假设两个节点有8个CPU，每个节点上只请求5个CPU。在这种情况下，默认调度器将认为这两个节点是等效的(假设其他一切都是相同的)。而且，可以对节点评分算法进行扩展，以有利于选用在给定时间段(例如，最近6小时)内平均实际利用率较低的节点。
如果节点1和节点2在给定时间段内的平均实际利用率相等，则只考虑平均利用率的评分算法无法区分这两个节点，可以随机选择其中一个节点，如图所示。



但是，通过查看节点上的历史数据和实际CPU利用率，可以清楚地看到节点2上的资源利用率比节点1上有更多的变化。因此，在高峰时间，其利用率更有可能超过总容量或目标利用率水平。节点1应该优先放置新Pod，以防止在高峰时间供应不足的风险，并保证新Pod有更好的性能。



除了根据实际使用情况进行有效调度外，还需要一个高级调度器来平衡高峰使用期间资源争用的风险。

# Trimaran:真实负载感知调度
为了最小化运行成本，调度程序可以了解其声明式资源分配模型和实际资源利用率之间的差距。与默认调度器相比，Pod可以更有效地打包在更少的节点中，默认调度器只考虑Pod请求和节点上的可分配资源(使用默认插件)。


有两种调度策略可用来增强OpenShift中的现有调度:Trimaran调度程序下的[TargetLoadPacking ](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran/targetloadpacking)和[Trimaran 调度器](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran)下的[LoadVariationRiskBalancing](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran/loadvariationriskbalancing)来解决这个问题。它目前支持像Prometheus, SignalFx和Kubernetes Metrics Server这样的度量提供商。插件为所有Pod QoS保证提供调度支持。

# 将节点利用率维持在指定级别:TargetLoadPacking Scheduler插件

许多用户都希望在CPU使用中保留一些空间，作为具有阈值的缓冲区，同时让节点的数量最小化。TargetLoadPacking插件就是为此设计的。TargetLoadPacking策略将工作负载打包到节点上，直到所有节点的利用率达到目标百分比。然后，它开始在所有节点中分配工作负载。使用TargetLoadPacking策略的好处是，所有正在运行的节点都保持一个目标利用率，因此没有节点利用率不足。但是，当集群几乎满时，它不会使特定节点过度超载，这为应用程序的负载变化留下了一些空间。



# 平衡利用率变化的风险:LoadVariationRiskBalancing Scheduler插件

众所周知，Kubernetes调度器依赖于请求的资源，而不是实际的资源使用情况。因此，这可能会导致集群中的争用和不平衡。扩展带有某种负载感知的默认调度器，试图解决这个问题。问题是调度程序应该采用负载变化的哪些方面。最常用的度量是资源利用率的平均或移动平均。虽然简单，但它没有捕捉到随时间变化的利用率。在这种调度策略中，考虑到资源争用的风险，其思想是提高平均与标准差的度量，从而使集群达到良好的均衡。LoadVariationRiskBalancing调度策略使用了一个简单而有效的优先级函数，该函数结合了节点利用率的平均值和标准偏差的度量。



# 系统设计与实现

我们开发了基于实时节点利用率值的[Trimaran 调度器](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran)，以有效地利用集群资源并节约成本。作为其中的一部分，我们开发了[Load Watcher](https://github.com/paypal/load-watcher)、[TargetLoadPacking插件](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran/targetloadpacking)和，并将[LoadVariationRiskBalancing插件](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran/loadvariationriskbalancing)其贡献给了开源社区。

下图显示了负载感知调度框架的设计。除了默认的Kubernetes调度器之外，还添加了一个负载监视器，它可以从指标提供程序(如Prometheus)定期检索、聚合和分析资源使用指标。它还缓存分析结果，并将这些结果公开给调度程序插件，以筛选和评分节点。默认情况下，负载监视器每分钟检索5分钟历史指标并缓存分析结果。但是，检索频率是可以配置的。

使用HTTP服务器提供数据查询是可选的，因为负载监视器也可以用分析的数据注释节点。但是，使用注释不像REST API那样灵活和可伸缩。假设需要一些高级预测分析来集成调度器插件。在这种情况下，更容易使用REST API将更多数据传递给调度程序插件。注释中要缓存的数据量是有限的。

Trimaran插件使用[Load Watcher](https://github.com/paypal/load-watcher)通过指标提供程序访问资源利用率数据。目前，负载监视器支持三个指标提供程序:[Kubernetes metrics Server](https://github.com/kubernetes-sigs/metrics-server)、[Prometheus Server](https://prometheus.io/)和[SignalFx](https://docs.signalfx.com/en/latest/integrations/agent/index.html)。

Trimaran插件使用负载监视器有两种模式:作为服务或作为库

1. 作为服务的负载监视器:在这种模式下，Trimaran插件使用集群中部署的负载监视器服务，如下图所示。定义负载监视服务端点需要一个watcherAddress配置参数。负载监视服务也可以部署在同一个调度程序Pod中。

2. 作为库的负载观察器:在这种模式下，Trimaran插件将负载观察器作为库嵌入，库反过来访问配置的指标提供程序。在本例中，我们有三个配置参数:metricProvider。类型,metricProvider。地址和metricProvider.token。

启用负载感知调度插件将导致与两个默认评分插件的冲突:“NodeResourcesLeastAllocated”和“NodeResourcesBalancedAllocation”评分插件。强烈建议禁用它们。
![在这里插入图片描述](https://img-blog.csdnimg.cn/04b8d6f9cff9403d9b86f29dedf4f042.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5ZWG5pyd,size_20,color_FFFFFF,t_70,g_se,x_16)
TargetLoadPacking插件的目的是根据节点的实际使用情况对节点进行评分。最终，所有节点的利用率可以维持在x%的水平。工作分为两个阶段，如下图所示。当集群中的大多数节点都处于空闲状态时，评分函数将优先考虑最大利用率低于x%的节点。当所有节点的利用率都在x%左右或以上时，评分函数将优先考虑利用率最低的节点。本质上，它将工作负载打包到节点上，直到所有节点的利用率达到x%左右，然后扩展工作负载以平衡节点上的负载。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2caf16fea265410daec3635e9d5e85c2.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5ZWG5pyd,size_20,color_FFFFFF,t_70,g_se,x_16)
仅使用平均资源利用率数字的负载感知调度器可能不够，因为节点上利用率的变化还会影响在该节点上运行的容器的性能。容器的资源可用性随时间的变化会影响包含此类容器的应用程序的执行和行为。因此，负载感知调度器应该同时考虑平均利用率和利用率的变化。上面提到了一个激励人心的例子，在这个例子中，平衡风险的负载感知调度程序应该倾向于放置新Pod的更稳定的节点。



[负载变化风险平衡插件](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran/loadvariationriskbalancing)(Load Variation Risk Balancing Plugin)基于均衡风险对节点进行评分，风险定义为节点间平均利用率和利用率变化的综合度量。支持CPU和内存资源。它依赖于负载监视器通过指标提供程序(如Prometheus和Kubernetes metrics)从节点收集资源利用率度量。



负载变化风险平衡插件是使用Kubernetes调度器框架作为使用Score扩展点的调度器插件来实现的。调度程序插件调用负载监视器作为获取关于CPU和内存利用率的平均和标准偏差的节点度量数据的方法。反过来，负载监视器将使用一个指标提供程序，例如Prometheus和Kubernetes metrics。资源风险因素定义为资源利用率的平均值和标准偏差的组合度量。对节点的CPU和内存资源进行风险评估。节点风险因子作为两个资源风险因子中较差的一个。节点评分与节点危险因素成负正比。



以 N1、N2、N3，3节点集群为例，每个集群4核，容量为 8 GB。LoadVariationRiskBalancing插件需要放置一个带有0.5核和 1 GB 请求资源的新Pod。从负荷观察器得到的平均和标准差指标为: N1 的平均负荷低，变化大;N2的平均负荷高，变化小; N3 的平均负荷中等，变化小。对3个节点进行评分，N3得分最高。从下图可以看出，在二维平均和标准差空间中，节点 N3 距离集群的风险平衡线最远。因此，调度程序选取 N3，将集群移向风险平衡的方向。
![在这里插入图片描述](https://img-blog.csdnimg.cn/ac3dcd6c49894332981c7abb52910e03.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5ZWG5pyd,size_20,color_FFFFFF,t_70,g_se,x_16)
<font size =1>就均衡集群而言，给定集群中所有节点的平均利用率和标准偏差利用率，负载感知调度器基本上有四种方法来实现平衡:</font>
1. 均衡负载:使节点之间的平均利用率相等，而不考虑变量。

2. 平衡可变性:使节点利用率的标准偏差相等，而不考虑平均值。

3. 平衡相对变异性:均衡节点之间的变异系数(定义为标准偏差与平均利用率的比值)。

4. 均衡风险:将节点间的风险因素(定义为利用率的平均值和标准差的加权和)进行均衡。

下图描述了平衡集群的四种方法:
![在这里插入图片描述](https://img-blog.csdnimg.cn/db6478797edd4f7d83124f4303ed1a13.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5ZWG5pyd,size_20,color_FFFFFF,t_70,g_se,x_16)
# 在OpenShift上部署Trimaran调度程序

OpenShift用户可以通过遵循[自定义](https://docs.openshift.com/container-platform/4.7/nodes/scheduling/nodes-custom-scheduler.html?extIdCarryOver=true&sc_cid=701f2000001Css5AAC)调度文档教程部署Trimaran调度程序作为OpenShift集群中的额外调度程序。



在下面的代码中，将部署一个负载感知调度器示例，该调度器正在运行树外[调度器插件](https://github.com/kubernetes-sigs/scheduler-plugins)。运行树外插件的调度器需要运行某个版本的调度器插件映像，并在调度器部署中将KubeSchedulerConfiguration传递给调度器二进制文件。例如，可以在[这里](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran)找到运行负载感知调度器所需的配置。



1. 为负载感知调度器配置插件,运行[TargetLoadPacking](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran/targetloadpacking)插件的负载感知调度程序可以配置如下:

2. 要将KubeSchedulerConfiguration信息传递给调度程序二进制文件，可以将其挂载为ConfigMap或挂载为主机上的本地文件。在这里，我们将其包装为ConfigMap trimaran-scheduler-config。
3. Yaml，在调度程序部署中作为卷挂载。已创建用于所有负载感知调度器资源的命名空间三体。

因此，为负载感知调度程序创建一个部署
```
    apiVersion: kubescheduler.config.k8s.io/v1beta1
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: false
    profiles:
    - schedulerName: trimaran-scheduler
      plugins:
        score:
          disabled:
          - name: NodeResourcesBalancedAllocation
          - name: NodeResourcesLeastAllocated
          enabled:
          - name: TargetLoadPacking
      pluginConfig:
      - name: TargetLoadPacking
        args:
          defaultRequests:
            cpu: "2000m"
          defaultRequestsMultiplier: "1"
          targetUtilization: 70
          metricProvider:
            type: Prometheus        ①
            address: https://       ②
            token:
```
配置度量Provider为Prometheus①。


从②中获取Prometheus路由URL。
```
oc get routes prometheus-k8s -n openshift-monitoring -ojson |jq ".status.ingress"|jq ".[0].host"|sed 's/"//g'

```
获得 Prometheus token 值③
```
> oc describe secret $(oc get secret -n openshift-monitoring|awk '{print $1}'|grep prometheus-k8s-token -m 1) -n openshift-monitoring|grep "token:"|cut -d: -f2|sed 's/^ *//g'
> oc create ns trimaran
> oc create -f trimaran-scheduler-config.yaml -n trimaran
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: trimaran-scheduler-config
  namespace: trimaran
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1beta1
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: false
    profiles:
    - schedulerName: trimaran-scheduler
      plugins:
        score:
          disabled:
          - name: NodeResourcesBalancedAllocation
          - name: NodeResourcesLeastAllocated
          enabled:
          - name: TargetLoadPacking
      pluginConfig:
      - name: TargetLoadPacking
        args:
          defaultRequests:
            cpu: "2000m"
          defaultRequestsMultiplier: "1"
          targetUtilization: 70
          metricProvider:
            type: Prometheus
            address: https://
            token: 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: trimaran-scheduler
  namespace: trimaran
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: trimaran
subjects:
- kind: ServiceAccount
  name: trimaran-scheduler
  namespace: trimaran
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trimaran-scheduler
  namespace: trimaran
  labels:
    app: trimaran-scheduler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trimaran-scheduler
  template:
    metadata:
      labels:
        app: trimaran-scheduler
    spec:
      serviceAccount: trimaran-scheduler
        volumes:
        - name: etckubernetes
          configMap:
            name: trimaran-scheduler-config
      containers:
        - name: trimaran-scheduler
          image: k8s.gcr.io/scheduler-plugins/kube-scheduler:v0.19.9 
          imagePullPolicy: Always
          args:
          - /bin/kube-scheduler
          - --config=/etc/kubernetes/config.yaml
          - -v=6
          volumeMounts:
          - name: etckubernetes
            mountPath: /etc/kubernetes 
```
# 总结



使用Trimaran调度器插件，用户可以实现在默认的Kubernetes调度器中没有实现的基本负载感知调度。Trimaran插件既可以平衡节点上的使用，使所有节点都达到一定百分比的利用率，也可以优先考虑过度使用Pod时风险较低的节点。Trimaran插件也可以与overcommitment一起使用，以获得更好的效率。



Trimaran插件可以进一步扩展。一个扩展可以为其他类型的资源，包括网络带宽和GPU使用启用多维箱打包。另一个扩展可以添加更多的ML或AI模型来预测Pod/节点的利用率，以实现更好的调度。对于LoadVariationRiskBalancing插件，可以进行扩展来考虑资源利用率的概率分布，而不仅仅是分布的第一个(平均)和第二个(方差)时刻。在这种情况下，可以根据分布的尾部来计算风险，这捕获了利用率高于配置值的概率。另外，可以应用预测技术来预测资源的使用情况，而不是仅仅依赖过去测量的使用情况。



欢迎在下列项目存储库中提供所有类型的贡献和评论:



* [Trimaran:负载感知的调度插件](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/trimaran)

* [负载监视器](https://github.com/paypal/load-watcher):一个集群范围的资源使用度量聚合和分析工具。
