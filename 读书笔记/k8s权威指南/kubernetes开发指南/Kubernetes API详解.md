#k8s  #note 

# Kubernetes API概叙
Kubernetes API是集群系统中的重要组成部分，Kubernetes中各种资源（对象）的数据都通过该API接口被提交到后端的持久化存储（etcd）中，Kubernetes集群中的各部件之间通过该API接口实现解耦合，同时Kubernetes集群中一个重要且便捷的管理工具kubectl也是通过访问该API接口实现其强大的管理功能的。Kubernetes API中的资源对象都拥有通用的元数据，资源对象也可能存在嵌套现象，比如在一个Pod里面嵌套多个Container。创建一个API对象是指通过API调用创建一条有意义的记录，该记录一旦被创建，Kubernetes就将确保对应的资源对象会被自动创建并托管维护。

在Kubernetes系统中，在大多数情况下，API定义和实现都符合标准的HTTP REST格式，比如通过标准的HTTP动词（POST、PUT、GET、DELETE）来完成对相关资源对象的查询、创建、修改、删除等操作。但同时，Kubernetes也为某些非标准的REST行为实现了附加的API接口，例如Watch某个资源的变化、进入容器执行某个操作等。另外，某些API接口可能违背严格的REST模式，因为接口返回的不是单一的JSON对象，而是其他类型的数据，比如JSON对象流或非结构化的文本日志数据等。
Kubernetes开发人员认为，任何成功的系统都会经历一个不断成长和不断适应各种变更的过程。因此，他们期望Kubernetes API是不断变更和增长的。同时，他们在设计和开发时，有意识地兼容了已存在的客户需求。通常，我们不希望将新的API资源和新的资源域频繁地加入系统，资源或域的删除需要一个严格的审核流程。

Kubernetes API文档官网为https://kubernetes.io/docs/reference，可以通过相关链接查看不同版本的API文档，例如Kubernetes 1.14版本的链接为https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14。

# Kubernetes API的扩展
随着Kubernetes的发展，用户对Kubernetes的扩展性也提出了越来越高的要求。从1.7版本开始，Kubernetes引入扩展API资源的能力，使得开发人员在不修改Kubernetes核心代码的前提下可以对Kubernetes API进行扩展，仍然使用Kubernetes的语法对新增的API进行操作，这非常适用于在Kubernetes上通过其API实现其他功能（例如第三方性能指标采集服务）或者测试实验性新特性（例如外部设备驱动）。
在Kubernetes中，所有对象都被抽象定义为某种资源对象，同时系统会为其设置一个API入口（API Endpoint），对资源对象的操作（如新增、删除、修改、查看等）都需要通过Master的核心组件API Server调用资源对象的API来完成。与APIServer的交互可以通过kubectl命令行工具或访问其RESTful API进行。每个API都可以设置多个版本，在不同的API URL路径下区分，例如“/api/v1”或“/apis/extensions/v1beta1”等。使用这种机制后，用户可以很方便地定义这些API资源对象（YAML配置），并将其提交给Kubernetes（调用RESTful API），来完成对容器应用的各种管理工作。
Kubernetes系统内置的Pod、RC、Service、ConfigMap、Volume等资源对象已经能够满足常见的容器应用管理要求，但如果用户希望将其自行开发的第三方系统纳入Kubernetes，并使用Kubernetes的API对其自定义的功能或配置进行管理，就需要对API进行扩展了。目前Kubernetes提供了以下两种机制供用户扩展API。
（1）使用CRD机制：复用Kubernetes的API Server，无须编写额外的APIServer。用户只需要定义CRD，并且提供一个CRD控制器，就能通过Kubernetes的API管理自定义资源对象了，同时要求用户的CRD对象符合API Server的管理规范。
（2）使用API聚合机制：用户需要编写额外的API Server，可以对资源进行更细粒度的控制（例如，如何在各API版本之间切换），要求用户自行处理对多个API版本的支持。
本节主要对CRD和API聚合这两种API扩展机制的概念和用法进行详细说明。

## 使用CRD扩展API资源
CRD本身只是一段声明，用于定义用户自定义的资源对象。但仅有CRD的定义并没有实际作用，用户还需要提供管理CRD对象的CRD控制器（CRD Controller），才能实现对CRD对象的管理。CRD控制器通常可以通过Go语言进行开发，并需要遵循Kubernetes的控制器开发规范，基于客户端库client-go进行开发，需要实现Informer 、ResourceEventHandler、Workqueue等组件具体的功能处理逻辑，详细的开发过程请参考官方示例（https://github.com/kubernetes/sample-controller）和client-go库（https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md）的详细说明。

## 1．创建CRD的定义
与其他资源对象一样，对CRD的定义也使用YAML配置进行声明。以Istio系统中的自定义资源VirtualService为例，配置文件crd-virtualservice.yaml的内容如下：
![[Pasted image 20220602162526.png]]
CRD定义中的关键字段如下。
（1）group：设置API所属的组，将其映射为API URL中“/apis/”的下一级目录，设置networking.istio.io生成的API URL路径为“/apis/networking.istio.io”。
（2）scope：该API的生效范围，可选项为Namespaced（由Namespace限定）和Cluster（在集群范围全局生效，不局限于任何Namespace），默认值为Namespaced。
（3）versions：设置此CRD支持的版本，可以设置多个版本，用列表形式表示。目前还可以设置名为version的字段，只能设置一个版本，在将来的Kubernetes版本中会被弃用，建议使用versions进行设置。如果该CRD支持多个版本，则每个版本都会在API URL“/apis/networking.istio.io ”的下一级进行体现，例如“/apis/networking.istio.io/v1 ”或“/apis/networking.istio.io/v1alpha3”等。每个版本都可以设置下列参数。
-  name：版本的名称，例如v1、v1alpha3等。
-  served：是否启用，在被设置为true时表示启用。
-  storage：是否进行存储，只能有一个版本被设置为true。

（4）names：CRD的名称，包括单数、复数、kind、所属组等名称的定义，可以设置如下参数。
- kind：CRD的资源类型名称，要求以驼峰式命名规范进行命名（单词的首字母都大写），例如VirtualService。
- listKind：CRD列表，默认被设置为< kind >List格式，例如VirtualServiceList。
- singular：单数形式的名称，要求全部小写，例如virtualservice。
- plural：复数形式的名称，要求全部小写，例如virtualservices。
- shortNames：缩写形式的名称，要求全部小写，例如vs。
- categories ： CRD所属的资源组列表。例如，VirtualService属于istio-io组和networking-istio-io组，用户通过查询istio-io组和networking-istio-io组，也可以查询到该CRD实例。

使用kubectl create命令完成CRD的创建：
![[Pasted image 20220602163120.png]]
在CRD创建成功后，由于本例的scope设置了Namespace限定，所以可以通过APIEndpoint“/apis/networking.istio.io/v1alpha3/namespaces/< namespace >/virtualservices/”管理该CRD资源。
用户接下来就可以基于该CRD的定义创建相应的自定义资源对象了。

## 2．基于CRD的定义创建自定义资源对象
基于CRD的定义，用户可以像创建Kubernetes系统内置的资源对象（如Pod）一样创建CRD资源对象。在下面的例子中，virtualservice-helloworld.yaml定义了一个类型为VirtualService的资源对象：
![[Pasted image 20220602163533.png]]
除了需要设置该CRD资源对象的名称，还需要在spec段设置相应的参数。在spec中可以设置的字段是由CRD开发者自定义的，需要根据CRD开发者提供的手册进行配置。这些参数通常包含特定的业务含义，由CRD控制器进行处理。
使用kubectl create命令完成CRD资源对象的创建：
![[Pasted image 20220602163728.png]]

## 3.CRD的高级特性
随着Kubernetes的演进，CRD也在逐步添加一些高级特性和功能，包括subresources子资源、校验（Validation）机制、自定义查看CRD时需要显示的列，以及finalizer预删除钩子。
### （1）CRD的subresources子资源Kubernetes从1.11版本开始，在CRD的定义中引入了名为subresources的配置，可以设置的选项包括status和scale两类。
◎ stcatus：启用/status路径，其值来自CRD的.status字段，要求CRD控制器能够设置和更新这个字段的值。
◎ scale：启用/scale路径，支持通过其他Kubernetes控制器（如HorizontalPodAutoscaler控制器）与CRD资源对象实例进行交互。用户通过kubectl scale命令也能对该CRD资源对象进行扩容或缩容操作，要求CRD本身支持以多个副本的形式运行。
下面是一个设置了subresources的CRD示例：
![[Pasted image 20220602164035.png]]
基于该CRD的定义，创建一个自定义资源对象my-crontab.yaml：
![[Pasted image 20220602164153.png]]
之后就能通过API Endpoint查看该资源对象的状态了：
![[Pasted image 20220602164301.png]]
并查看该资源对象的扩缩容（scale）信息：
![[Pasted image 20220602164314.png]]
用户还可以使用kubectl scale命令对Pod的副本数量进行调整，例如：
![[Pasted image 20220602164326.png]]

### (2) CRD的校验(Validation)机制
Kubernetes从1.8版本开始引入了基于OpenAPI v3 schema或validatingadmissionwebhook的校验机制，用于校验用户提交的CRD资源对象配置是否符合预定义的校验规则。该机制到Kubernetes 1.13版本时升级为Beta版。要使用该功能，需要为kube-apiserver服务开启--feature-gates=CustomResourceValidation=true特性开关。
下面的例子为CRD定义中的两个字段（cronSpec和replicas）设置了校验规则：
![[Pasted image 20220602165342.png]]
校验规则如下。
◎ spec.cronSpec：必须为字符串类型，并且满足正则表达式的格式。
◎ spec.replicas：必须将其设置为1～10的整数。
对于不符合要求的CRD资源对象定义，系统将拒绝创建。例如，下面的my-crontab.yaml示例违反了CRD中validation设置的校验规则，即cronSpec没有满足正则表达式的格式，replicas的值大于10：
![[Pasted image 20220602165458.png]]
创建时，系统将报出validation失败的错误信息

### （3）自定义查看CRD时需要显示的列
从Kubernetes 1.11版本开始，通过kubectl get命令能够显示哪些字段由服务端（API Server）决定，还支持在CRD中设置需要在查看（get）时显示的自定义列，在spec.additionalPrinterColumns字段设置即可。

### （4）Finalizer（CRD资源对象的预删除钩子方法）
Finalizer设置的方法在删除CRD资源对象时进行调用，以实现CRD资源对象的清理工作。
在下面的例子中为CRD“CronTab”设置了一个finalizer（也可以设置多个），其值为URL“finalizer.stable.example.com”：
![[Pasted image 20220602165801.png]]
在用户发起删除该资源对象的请求时，Kubernetes不会直接删除这个资源对象，而是在元数据部分设置时间戳“metadata.deletionTimestamp”的值，标记为开始删除该CRD对象。然后控制器开始执行finalizer定义的钩子方法“finalizer.stable.example.com”进行清理工作。对于耗时较长的清理操作，还可以设置metadata.deletionGracePeriodSeconds超时时间，在超过这个时间后由系统强制终止钩子方法的执行。在控制器执行完钩子方法后，控制器应负责删除相应的finalizer。当全部finalizer都触发控制器执行钩子方法并都被删除之后，Kubernetes才会最终删除该CRD资源对象。
## 4．小结
CRD极大扩展了Kubernetes的能力，使用户像操作Pod一样操作自定义的各种资源对象。CRD已经在一些基于Kubernetes的第三方开源项目中得到广泛应用，包括CSI存储插件、Device Plugin（GPU驱动程序）、Istio（Service Mesh管理）等，已经逐渐成为扩展Kubernetes能力的标准。

# 使用API聚合机制扩展API资源
API聚合机制是Kubernetes 1.7版本引入的特性，能够将用户扩展的API注册到kube-apiserver上，仍然通过API Server的HTTP URL对新的API进行访问和操作。为了实现这个机制，Kubernetes在kube-apiserver服务中引入了一个API聚合层（API Aggregation Layer），用于将扩展API的访问请求转发到用户服务的功能。
设计API聚合机制的主要目标如下。
◎ 增加API的扩展性：使得开发人员可以编写自己的API Server来发布他们的API，而无须对Kubernetes核心代码进行任何修改。
◎ 无须等待Kubernetes核心团队的繁杂审查：允许开发人员将其API作为单独的API Server发布，使集群管理员不用对Kubernetes核心代码进行修改就能使用新的API，也就无须等待社区繁杂的审查了。
◎ 支持实验性新特性API开发：可以在独立的API聚合服务中开发新的API，不影响系统现有的功能。
◎ 确保新的API遵循Kubernetes的规范：如果没有API聚合机制，开发人员就可能会被迫推出自己的设计，可能不遵循Kubernetes规范。
总的来说，API聚合机制的目标是提供集中的API发现机制和安全的代理功能，将开发人员的新API动态地、无缝地注册到Kubernetes API Server中进行测试和使用。
下面对API聚合机制的使用方式进行详细说明。
## 1．在Master的API Server中启用API聚合功能
为了能够将用户自定义的API注册到Master的API Server中，首先需要配置kube-apiserver服务的以下启动参数来启用API聚合功能。
◎ --requestheader-client-ca-file=/etc/kubernetes/ssl_keys/ca.crt：客户端CA证书。
◎ --requestheader-allowed-names=：允许访问的客户端common names列表，通过header中--requestheader-username-headers参数指定的字段获取。客户端common names的名称需要在client-ca-file中进行设置，将其设置为空值时，表示任意客户端都可访问。
◎ --requestheader-extra-headers-prefix=X-Remote-Extra-：请求头中需要检查的前缀名。
◎ --requestheader-group-headers=X-Remote-Group：请求头中需要检查的组名。
◎ --requestheader-username-headers=X-Remote-User：请求头中需要检查的用户名。
◎ --proxy-client-cert-file=/etc/kubernetes/ssl_keys/kubelet_client.crt ：在请求期间验证Aggregator的客户端CA证书。
◎ --proxy-client-key-file=/etc/kubernetes/ssl_keys/kubelet_client.key：在请求期间验证Aggregator的客户端私钥。
如果kube-apiserver所在的主机上没有运行kube-proxy，即无法通过服务的ClusterIP进行访问，那么还需要设置以下启动参数：
```bash
--enable-aggregator-routing=true
```

## 2．注册自定义APIService资源
在启用了API Server的API聚合功能之后，用户就能将自定义API资源注册到Kubernetes Master的API Server中了。用户只需配置一个APIService资源对象，就能进行注册了。APIService示例的YAML配置文件如下：
![[Pasted image 20220802143250.png]]
在这个APIService中设置的API组名为custom.metrics.k8s.io，版本号为v1beta1，这两个字段将作为API路径的子目录注册到API路径“/apis/”下。注册成功后，就能通过Master API路径“/apis/custom.metrics.k8s.io/v1beta1”访问自定义的API Server了。
在service段中通过name和namespace设置了后端的自定义API Server，本例中的服务名为custom-metrics-server，命名空间为custom-metrics。
通过kubectl create命令将这个APIService定义发送给Master，就完成了注册操作。
之后，通过Master API Server对“/apis/custom.metrics.k8s.io/v1beta1”路径的访问都会被API聚合层代理转发到后端服务custom-metrics-server.custom-metrics.svc上了。
## 3．实现和部署自定义API Server
仅仅注册APIService资源还是不够的，用户对“/apis/custom.metrics.k8s.io/v1beta1”路径的访问实际上都被转发给了custom-metrics-server.custom-metrics.svc服务。这个服务通常能以普通Pod的形式在Kubernetes集群中运行。当然，这个服务需要由自定义API的开发者提供，并且需要遵循Kubernetes的开发规范，详细的开发示例可以参考官方给出的示例（https://github.com/kubernetes/sample-apiserver）。
下面是部署自定义API Server的常规操作步骤。
（1）确保APIService API已启用，这需要通过kube-apiserver的启动参数--runtime-config进行设置，默认是启用的。
（2）建议创建一个RBAC规则，允许添加APIService资源对象，因为API扩展对整个Kubernetes集群都生效，所以不推荐在生产环境中对API扩展进行开发或测试。
（3）创建一个新的Namespace用于运行扩展的API Server。
（4）创建一个CA证书用于对自定义API Server的HTTPS安全访问进行签名。
（5）创建服务端证书和秘钥用于自定义API Server的HTTPS安全访问。服务端证书应该由上面提及的CA证书进行签名，也应该包含含有DNS域名格式的CN名称。
（6）在新的Namespace中使用服务端证书和秘钥创建Kubernetes Secret对象。
（7）部署自定义API Server实例，通常可以以Deployment形式进行部署，并且将之前创建的Secret挂载到容器内部。该Deployment也应被部署在新的Namespace中。
（8）确保自定义的API Server通过Volume加载了Secret中的证书，这将用于后续的HTTPS握手校验。
（9）在新的Namespace中创建一个Service Account对象。
（10）创建一个ClusterRole用于对自定义API资源进行操作。
（11）使用之前创建的ServiceAccount为刚刚创建的ClusterRole创建一个ClusterRolebinding。
（12）使用之前创建的ServiceAccount为系统ClusterRole“system:auth-delegator”创建一个ClusterRolebinding，以使其可以将认证决策代理转发给Kubernetes核心API Server。
（13）使用之前创建的ServiceAccount为系统Role“extension-apiserver-authentication-reader”创建一个Rolebinding，以允许自定义API Server访问名为“extension-apiserver-authentication”的系统ConfigMap。
（14）创建APIService资源对象。
（15）访问APIService提供的API URL路径，验证对资源的访问能否成功。
下面以部署Metrics Server为例，说明一个聚合API的实现方式。
随着API聚合机制的出现，Heapster也进入弃用阶段，逐渐被Metrics Server替代。Metrics Server通过聚合API提供Pod和Node的资源使用数据，供HPA控制器、VPA控制器及kubectl top命令使用。Metrics Server的源码可以在GitHub代码库（https://github.com/kubernetes-incubator/metrics-server）找到，在部署完成后，Metrics Server将通过Kubernetes核心API Server的“/apis/metrics.k8s.io/v1beta1”路径提供Pod和Node的监控数据。
首先，部署Metrics Server实例，在下面的YAML配置中包含一个ServiceAccount、一个Deployment和一个Service的定义：
![[Pasted image 20220802144352.png]]
![[Pasted image 20220802144408.png]]
![[Pasted image 20220802144419.png]]
然后，创建Metrics Server所需的RBAC权限配置：
![[Pasted image 20220802144507.png]]
![[Pasted image 20220802144624.png]]
![[Pasted image 20220802144637.png]]
![[Pasted image 20220802144649.png]]
最后，定义APIService资源，主要设置自定义API的组（group）、版本号（version）及对应的服务（metrics-server.kube-system）：
![[Pasted image 20220802144709.png]]
在所有资源都成功创建之后，在命名空间kube-system中会看到新建的metrics-server Pod。
通过Kubernetes Master API Server的URL“/apis/metrics.k8s.io/v1beta1”就能查询到Metrics Server提供的Pod和Node的性能数据了：
![[Pasted image 20220802144736.png]]
![[Pasted image 20220802144750.png]]
![[Pasted image 20220802144800.png]]
