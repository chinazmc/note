#k8s 

**“计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决”**  

  

软件行业没有什么问题是不能通过增加一个中间层来解决的，如果不能，那就加两个。

  

01

—

前言

  

那为什么我们需要编写k8s的控制器？

  

它是不是另外一种形式的中间层？  

  

Kubernetes已经成为了云原生事实标准，写控制器代码让程序自动化完成任务，已经是一个开发人员的基本要求。

  

在代码领域，有非常多的设计模式。

  

每一种设计模式的初衷都是为了代码尽可能的精简、更多的被复用。

  

自动生成代码也是基于这种思想而来，这也是为什么Kubernetes如此受开发者青睐的原因之一。  

  

好比是go也同样有go generate可以自动生成代码一样，也是基于这个思想。  

  

对于一个Kubernetes开发者来讲，不同于传统程序开发，你不用关注程序本身的启动、并发时选举、随意定义的API格式等。你只用关注你的业务逻辑就好了，其余的事情框架帮你已经全部完成。

  

Kubebuilder就是官方社区自动生成脚手架代码的二进制工具。  

  

02

—

安装  

Kubebuilder安装很简单，直接通过github release页面下载对应平台的二进制命令即可。

  

https://github.com/kubernetes-sigs/kubebuilder/releases

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/rsqlJmRz97xYlVDB4OHBficwYIf7bMYdlvp2AB7Ta65T4zaCWfiaibYBc09icmSj1XCLGE4HeDCg8EIjCbQwcaficWA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  

可以看到提供了 arm x86 和 macOS，但是windows的怎么办？

  

这也是windows开发者刚开始的烦恼。

  

其实最简单的方法就是在Windows上安装WSL，可以无缝使用x86的命令来执行。

  

如果喜欢折腾，也可以自己拿源码自己编译一个二进制，但是确实没有必要。  

  

  

下载下来后，把kubebuilder命令加下执行权限，并放到系统变量中。  

  

```
chmod +x kubebuilder
```

  

此时，我们已经可以执行这个命令了。

  

03

—  

初始化  

  

1. 在使用kubebuilder的时候，首先，我们要有一个空目录：  

  

```
mkdir /tmp/test
```

  

2. 然后，我们需要go mod init初始化我们的项目：  

  

```
go mod init apiserver
```

  

3. 此时会生成go.mod文件，初始kubuilder项目  

  

```
kubebuilder init --domain ""
```

注意：这里用的是domain是空，否则所有的资源创建后都会带这个domain  

  

 执行结果如下：：  

  

```
[root@i-b2e6autw test]# kubebuilder init --domain ""
```

  

生成的项目目录树如下：  

  

```
[root@i-b2e6autw test]# tree .
```

04

—  

创建CRD及Controller

  

  

**什么是 CRD？**   

  

它是Custom Resource Defination的缩写，意思就是用户自定义资源。  

  

为什么需要它？难道Kubernetes提供的内置资源不能满足我们的要求吗？

  

确实。内置的资源模型太少了。而且都是固定的格式，不能自己定义。

  

比如：你要使用Deployment，你就要按标准的deloyment定义的每个字段，来写固定的数据。

  

如果我们要自己定义一些字段，来保存我们自己的业务字段，就需要我们自定义资源。

  

生成CRD  

  

```
 kubebuilder create api --group test.io --version v1alpha1 --kind Traffic
```

  

  

执行过程中会询问是否创建 Resource及Controller，我们都选择是。  

  

```
[root@i-b2e6autw test]#   kubebuilder create api --group test.io --version v1alpha1 --kind Traffic
```

  

这样就生成了CRD及controller文件。  

  

```
[root@i-b2e6autw test]# tree api
```

  

05

—  

项目构建

  

上面，我们已经创建Traffic的crd及controller文件。  

  

此时，我们应该要去修改下CRD文件，来自定义我们的字段：

  

```
 # ./api/v1alpha1/traffic_types.go
```

  

在上面的文件中，我们修改的api中的CRD中的字段，待会儿我们可以直接使用这个字段。

  

生成manifest，什么是Manifest?  

  

简单来讲就是生成CRD的yaml，只有apply这个CRD yaml后，我们才可以创建对应CR资源。  

  

CRD mainfest定义的是每个字段数据格式，使用的OpenAPI v3版本。

  

```
 make manifests
```

  

执行完后，可以看到生成了CRD文件 

  

```
# ls config/crd/bases/test.io.my.domain_traffics.yaml
```

  

此时，我们可以直接apply这个crd，就可以创建对应的CR文件了。  

  

  

```
[root@i-b2e6autw test]# kubectl apply -f config/crd/bases/test.io_traffics.yaml
```

  

检查CRD已经成功创建：  

  

```
[root@i-b2e6autw test]# kubectl get crd | grep -i traffic
```

  

我们现在可以直接创建一个对应的CR，看下效果。

  

apply 一下这个yaml示例：

  

```
apiVersion: test.io/v1alpha1
```

  

会提示创建成功：  

  

```
[root@i-b2e6autw test]# kubectl apply -f  config/samples/test.io_v1alpha1_traffic.yaml
```

  

  

获取资源CR：

  

```
[root@i-b2e6autw test]# kubectl get traffic -oyaml
```

  

  

至此，我们已经生成并apply了CRD文件，且功创建了CR。

  

  

  

  

  

06

—  

编写控制器业务逻辑  

  

如果只是为了保存一下自定义资源到ETCD，我们的目的其实已经完成。  

  

但是如果我们要使用这些数据，我们还要编写控制器业务逻辑。

  

```
# controllers/traffic_controller.go
```

  

上面的业务逻辑很简单，就是去Get下资源，并打印下名字而已。  

  

现在我们需要把这个项目跑起来。

  

  

07

—  

打包编译

  

至此，我们需要把这个项目在kubernetes上跑起来。  

  

首先要打镜像：  

  

```
[root@i-b2e6autw test]# make docker-build
```

通过上面的细节，我们知道它会生成 controller:latest镜像  

  

我们把镜像推送到自己仓库：

  

```
# docker tag controller:latest zackzhangkai/controller-test:latest
```

  

部署

  

# make deploy`...``cd config/manager && /tmp/test/bin/kustomize edit set image controller=controller:latest``/tmp/test/bin/kustomize build config/default | kubectl apply -f -``namespace/test-system created``customresourcedefinition.apiextensions.k8s.io/traffics.test.io configured``serviceaccount/test-controller-manager created``role.rbac.authorization.k8s.io/test-leader-election-role created``clusterrole.rbac.authorization.k8s.io/test-manager-role created``clusterrole.rbac.authorization.k8s.io/test-metrics-reader created``clusterrole.rbac.authorization.k8s.io/test-proxy-role created``rolebinding.rbac.authorization.k8s.io/test-leader-election-rolebinding created``clusterrolebinding.rbac.authorization.k8s.io/test-manager-rolebinding created``clusterrolebinding.rbac.authorization.k8s.io/test-proxy-rolebinding created``configmap/test-manager-config created``service/test-controller-manager-metrics-service created``deployment.apps/test-controller-manager created``...`

可以看到，它把rbac文件及Deplyment等资源apply了。  

  

查看：  

```
# kubectl -n test-system get all
```

  

如果deloyment没有正确运行，手动修改下镜像 为zackzhangkai/controller:latest即可正常。  

  

此时可以看到日志正常打印。  

  

至此，我们已经完整的写完了operator代码。

  

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

  


# Reference
https://mp.weixin.qq.com/s/pIJUoaDnXMIb7p9IvMdcvw

相关代码存放在：

https://github.com/zackzhangkai/showcase

  

