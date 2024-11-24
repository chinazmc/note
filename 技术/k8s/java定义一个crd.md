#k8s 

## 安装Kubernetes【Minikube】

minikube为开发或者测试在本地启动一个节点的kubernetes集群，minikube打包了和配置一个linux虚拟机、docker与kubernetes组件。

Kubernetes集群是由Master和Node组成的，Master用于进行集群管理，Node用于运行Pod等workload。而minikube是一个Kubernetes集群的最小集。

### 1、安装virtualbox

[https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)

### 2、安装minikube

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

### 3、启用dashboard（web console）【可选】

```bash
minikube addons enable dashboard
```

开启dashboard

### 4、启动minikube

minikube start

start之后可以通过minikube status来查看状态，如果minikube和cluster都是Running，则说明启动成功。

### 5、查看启动状态

kubectl get pods


# CRD【CustomResourceDefinition】

CRD是Kubernetes为提高可扩展性，让开发者去自定义资源（如Deployment，StatefulSet等）的一种方法。

Operator=CRD+Controller。

CRD仅仅是资源的定义，而Controller可以去监听CRD的CRUD事件来添加自定义业务逻辑。

关于CRD有一些链接先贴出来：

-   官方文档：[https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/)
-   CRD Yaml的Schema：[https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#customresourcedefinition-v1beta1-apiextensions-k8s-io](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#customresourcedefinition-v1beta1-apiextensions-k8s-io)
-   [https://kubernetes.feisky.xyz/cha-jian-kuo-zhan/api/customresourcedefinition](https://kubernetes.feisky.xyz/cha-jian-kuo-zhan/api/customresourcedefinition)
-   [https://sq.163yun.com/blog/article/174980128954048512](https://sq.163yun.com/blog/article/174980128954048512)
-   [https://book.kubebuilder.io/](https://book.kubebuilder.io/)

如果说只是对CRD实例进行CRUD的话，不需要Controller也是可以实现的，只是只有数据，没有针对数据的操作。

一个CRD的yaml文件示例：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  version: v1beta1
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

通过kubectl create -f crd.yaml可以创建一个CRD。

kubectl get CustomResourceDeinitions可以获取创建的所有CRD。

![img](https://cdn.nlark.com/lark/0/2018/png/115178/1544523122252-4e86b6d5-679d-4aa9-8846-007b52869184.png)

然后可以通过kubectl create -f my-crontab.yaml可以创建一个CRD的实例：

```yaml
apiVersion: "stable.example.com/v1beta1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

![img](https://cdn.nlark.com/lark/0/2018/png/115178/1544523365960-63cb2113-af95-462b-a0ec-8be2cc4e8b80.png)

## Controller【Fabric8】

如何去实现一个Controller呢？

可以使用Go来实现，并且不论是参考资料还是开源支持都非常好，推荐有Go语言基础的优先考虑用[client-go](https://github.com/kubernetes/client-go)来作为Kubernetes的客户端，用[KubeBuilder](https://github.com/kubernetes-sigs/kubebuilder)来生成骨架代码。一个官方的Controller示例项目是[sample-controller](https://github.com/kubernetes/sample-controller)。

对于Java来说，目前Kubernetes的JavaClient有两个，一个是Jasery，另一个是[Fabric8](https://github.com/fabric8io/kubernetes-client)。后者要更好用一些，因为对Pod、Deployment都有DSL定义，而且构建对象是以Builder模式做的，写起来比较舒服。

Fabric8的资料目前只有[https://github.com/fabric8io/kubernetes-client](https://github.com/fabric8io/kubernetes-client)，注意看目录下的examples。

这些客户端本质上都是通过REST接口来与Kubernetes API Server通信的。

Controller的逻辑其实是很简单的：监听CRD实例（以及关联的资源）的CRUD事件，然后执行相应的业务逻辑

# MyDeployment

基于Fabric8来开发一个较为完整的Controller的示例目前我在网络上是找不到的，下面MyController的实现也是一步步摸索出来的，难免会有各种问题=.=，欢迎大佬们捉虫。

### 用例

代码在[https://github.com/songxinjianqwe/deployment-controller](https://github.com/songxinjianqwe/deployment-controller)。

MyDeployment是用来模拟Kubernetes提供的Deployment的简化版实现，目前可以做到以下功能：

-   启动后自动创建出一个MyDeployment的CRD
    -   【触发】启动应用
    -   【期望】可以看到创建出来的CRD
    -   【测试】kubectl get CustomResourceDefinition -o yaml
-   创建一个MyDeployment: Nginx实例
    -   【触发】kubectl create -f my-deployment-instance.yaml
    -   【期望】可以看到级联创建出来的3个pod
    -   【测试】kubectl get pods
-   手工删掉一个pod
    -   【触发】kubectl delete pods $pod_name
    -   【期望】pod被重建
    -   【测试】kubectl get pods -w
-   暴露一个服务
    -   【触发】kubectl create -f my-deployment-service.yaml
    -   【期望】可以通过curl来访问nginx服务
    -   【测试】minikube service my-nginx-app –url 然后 curl
-   更新镜像
    -   【触发】kubectl replace -f my-deployment-instance-update-image-1.9.1.yaml
    -   【期望】pod的nginx版本被更新为1.9.1
    -   【测试】kubectl get pods -o yaml
-   扩容
    -   【触发】kubectl replace -f my-deployment-instance-update-scaleup-1.9.1.yaml
    -   【期望】pod被扩容到5个
    -   【测试】kubectl get pods
-   缩容
    -   【触发】kubectl replace -f my-deployment-instance-update-scaledown-1.9.1.yaml
    -   【期望】pod被缩容到2个
    -   【测试】kubectl get pods
-   扩容并更新镜像
    -   【触发】kubectl replace -f my-deployment-instance-update-image-and-scaleup-1.14.yaml
    -   【期望】pod被扩容5个，且nginx版本被更新为1.14
    -   【测试】kubectl get pods 然后 kubectl get pods -o yaml
-   删除一个MyDeployment
    -   【触发】kubectl delete mydeployments my-nginx-app
    -   【期望】MyDeployment被删掉，并且关联的pod也被级联删掉
    -   【测试】kubectl get mydeployments 然后 kubectl get pods

此外还有一些功能尚未开发，其中状态是非常重要的，很可惜时间不够没有开发完成：

-   查看状态(TODO)
-   回滚(TODO)
-   状态更新【current，update-to-date，available】(TODO)
-   describe EVENTS(TODO)

### 运行

1、搭建好上面的minikube环境后

2、拉下deployment-controller的代码，是一个SpringBoot工程。

3、启动kube-proxy

kubectl proxy –port=12000

这一步是为了绕开Kubernetes API Server的权限认证。开启proxy之后就可以通过localhost:12000来连接Kubernetes了。

4、运行DeploymentControllerApplication的main方法即可。

5、此时可以根据上述用例来进行测试。

## MyDeployment实现

### 项目工程结构

![img](https://cdn.nlark.com/lark/0/2018/png/115178/1544529272311-c8aa028d-52ac-4cd9-800e-c6dcb5e5331e.png)

### CRD定义

按照Fabric8的逻辑，定义一个CRD需要至少定义以下内容：

-   CustomResourceDefinition，需要继承CustomResource，CRD资源定义
-   CustomResourceDefinitionList，需要继承CustomResourceList，CRD资源列表
-   CustomResourceDefinitionSpec，需要实现KubernetesResource接口，CRD资源的Spec
-   DoneableCustomResourceDefinition，需要继承CustomResourceDoneable，CRD资源的修改Builder
-   【可选】CustomResourceDefinitionStatus（需要说明一点的是，CRD支持使用SubResource，包括scale和status，在1.11+之后可以直接使用，在1.10中需要修改API Server的启动参数来启用；minikube的最新版本是可以支持到Kubernetes的1.10的）

在CRD定义中通常是需要持有一个Spec的【注意，上面提到的所有类定义持有的成员变量最好都加一个@JsonProperty注解，不加的话在get资源时对JSON反序列化时用到的名字就是属性名了】

下面是基于Fabric8的API构建出了一个CRD，之后可以调用API将其创建到Kubernetes，效果和kubectl create -f 是一样的。

但Controller需要做到的是启动时自动创建一个CRD出来，所以用kubectl创建不够自动化。
```java
public static final String CRD_GROUP = "cloud.alipay.com";
public static final String CRD_SINGULAR_NAME = "mydeployment";
public static final String CRD_PLURAL_NAME = "mydeployments";
public static final String CRD_NAME = CRD_PLURAL_NAME + "." + CRD_GROUP;
public static final String CRD_KIND = "MyDeployment";
public static final String CRD_SCOPE = "Namespaced";
public static final String CRD_SHORT_NAME = "md";
public static final String CRD_VERSION = "v1beta1";
public static final String CRD_API_VERSION = "apiextensions.k8s.io/" + CRD_VERSION;    

public static CustomResourceDefinition MY_DEPLOYMENT_CRD = new CustomResourceDefinitionBuilder()
            .withApiVersion(CRD_API_VERSION)
            .withNewMetadata()
            .withName(CRD_NAME)
            .endMetadata()

            .withNewSpec()
            .withGroup(CRD_GROUP)
            .withVersion(CRD_VERSION)
            .withScope(CRD_SCOPE)
            .withNewNames()
            .withKind(CRD_KIND)
            .withShortNames(CRD_SHORT_NAME)
            .withSingular(CRD_SINGULAR_NAME)
            .withPlural(CRD_PLURAL_NAME)
            .endNames()
            .endSpec()

            .withNewStatus()
            .withNewAcceptedNames()
            .addToShortNames(new String[]{"availableReplicas", "replicas", "updatedReplicas"})
            .endAcceptedNames()
            .endStatus()
            .build();
```
### Controller

入口处需要去为我们的CRD注册一个反序列化器。

入口处
```java
static {
    KubernetesDeserializer.registerCustomKind(MyDeployment.CRD_GROUP + "/" + MyDeployment.CRD_VERSION, MyDeployment.CRD_KIND, MyDeployment.class);
}

/**
 * 入口
 */
public void run() {
    // 创建CRD
    CustomResourceDefinition myDeploymentCrd = createCrdIfNotExists();
    // 监听Pod的事件
    watchPod();
    // 监听MyDeployment的事件
    watchMyDeployment(myDeploymentCrd);
}
```
watchPod是监听MyDeployment所管理的Pod的CRUD事件。
```java
private void watchPod() {
    delegate.client().pods().watch(new Watcher<Pod>() {
        @Override
        public void eventReceived(Action action, Pod pod) {
            // 如果是被MyDeployment管理的Pod
            if(pod.getMetadata().getOwnerReferences().stream().anyMatch(ownerReference -> ownerReference.getKind().equals(MyDeployment.CRD_KIND))) {
                unifiedPodWatcher.eventReceived(action, pod);
            }
        }

        @Override
        public void onClose(KubernetesClientException e) {
            log.error("watching pod {} caught an exception {}", e);
        }
    });
}
```
UnifiedPodWatcher是处理了所有Pod的事件，然后在收到事件时去通知Pod事件的订阅者，这里用到了一个观察者模式。
```java
@Component
public class UnifiedPodWatcher {
    @Autowired
    private List<PodAddedWatcher> podAddedWatchers;
    @Autowired
    private List<PodModifiedWatcher> podModifiedWatchers;
    @Autowired
    private List<PodDeletedWatcher> podDeletedWatchers;

    /**
     * 将Pod事件统一收到此处
     * @param action
     * @param pod
     */
    public void eventReceived(Watcher.Action action, Pod pod) {
        log.info("Thread {}: PodWatcher: {} =>  {}, {}", Thread.currentThread().getId(), action, pod.getMetadata().getName(), pod);
        switch (action) {
            case ADDED:
                podAddedWatchers.forEach(watcher -> watcher.onPodAdded(pod));
                break;
            case MODIFIED:
                podModifiedWatchers.forEach(watcher -> watcher.onPodModified(pod));
                break;
            case DELETED:
                podDeletedWatchers.forEach(watcher -> watcher.onPodDeleted(pod));
                break;
            default:
                break;
        }
    }
}
```
Fabric8的watcher是在代码层面不会限制在多个地方去监听同一个对象的，但经粗略测试，多处监听只有在第一个地方会收到回调；从可维护性角度来说，散落在各个地方的watcher代码也是不够优雅的。所以将watcher统一收口到一个地方，然后在这个地方下发事件。

然后MyDeployment的监听也是比较类似的：
```java
private void watchMyDeployment(CustomResourceDefinition myDeploymentCrd) {
    MixedOperation<
				    MyDeployment, 
				    MyDeploymentList, 
				    DoneableMyDeployment, 
				    Resource<MyDeployment, DoneableMyDeployment>
				> myDeploymentClient = 
				    delegate.client().customResources(
						    myDeploymentCrd, 
						    MyDeployment.class, 
						    MyDeploymentList.class, 
						    DoneableMyDeployment.class);
    myDeploymentClient.watch(new Watcher<MyDeployment>() {
        @Override
        public void eventReceived(Action action, MyDeployment myDeployment) {
            log.info("myDeployment: {} => {}" , action , myDeployment);
	if(myDeploymentHandlers.containsKey(
			MyDeploymentActionHandler.RESOURCE_NAME + action.name())
	) {
            myDeploymentHandlers.get(MyDeploymentActionHandler.RESOURCE_NAME + action.name()).handle(myDeployment);
            }
        }

        @Override
        public void onClose(KubernetesClientException e) {
            log.error("watching myDeployment {} caught an exception {}", e);
        }
    });
}
```

## 创建MyDeployment的实现逻辑

创建MyDeployment时会创建出replicas个Pod出来。此处逻辑在MyDeploymentAddedHandler实现。

```go
@Component(value = MyDeploymentActionHandler.RESOURCE_NAME + CrdAction.ADDED)
@Slf4j
public class MyDeploymentAddedHandler implements MyDeploymentActionHandler {
    @Autowired
    private KubeClientDelegate delegate;

    @Override
    public void handle(MyDeployment myDeployment) {
        log.info("{} added", myDeployment.getMetadata().getName());
        // TODO 当第一次启动项目时，现存的MyDeployment会回调一次Added事件，这里会导致重复创建pod【可通过status解决】,目前解法是去查一下现存的pod[不可靠]
        // 有可能pod的状态还没来得及置为not ready
        int existedReadyPodNumber = delegate.client().pods().inNamespace(myDeployment.getMetadata().getNamespace()).withLabelSelector(myDeployment.getSpec().getLabelSelector()).list().getItems()
                .stream().filter(UnifiedPodWatcher::isPodReady).collect(Collectors.toList()).size();
        Integer replicas = myDeployment.getSpec().getReplicas();
        for (int i = 0; i < replicas - existedReadyPodNumber; i++) {
            Pod pod = myDeployment.createPod();
            log.info("Thread {}:creating pod[{}]: {} , {}", Thread.currentThread().getId(), i, pod.getMetadata().getName(), pod);
            delegate.client().pods().create(pod);
       }
    }
}
```

这里需要解释一下为什么创建pod的数量需要减去existedReadyPodNumber。经观察发现，如果现存有CRD实例，然后启动Controller，会立即收到CRD的added事件，即使Pod都是健康的，这会导致创建出双倍的Pod。所以这里需要判断一下现存的Pod数量，如果够了，就不去创建了。

但是这又会引入一个问题，假如我将MyDeployment删掉了，此时会级联删除关联的Pod，在没来得及删掉之前，又去创建一个新的MyDeployment，这时候会发现现存的Pod数量并非为0，所以新建的Pod数量就不能达到replicas。解决办法是去判断一下Pod的状态，如果是NotReady，那么就不算是正常的Pod。

但这种解决思路还是有问题，在删掉MyDeployment之后不会立即将Pod状态置为NotReady，需要一定时间，在这段时间内如果创建MyDeployment，那么还是有可能会出现少创建Pod的情况。

目前还没有什么无BUG的思路。

创建一个Pod的实现，将这段代码放到了MyDeployment中，以表示从属关系（代码也好写一些）。

如果Pod是从属于某个MyDeployment，那么我们应该将OwnerReference传入；

Pod的name必须以MyDeployment的name为前缀，后面加上唯一ID；

Pod的spec必须与MyDeployment的spec中的pod template一致；

Pod的labels中包含MyDeployment的label，并且要加上一个pod-template哈希值，以在更新资源时判断pod template是否改变，如果没有变化，则不触发modified事件。

> **Note:** A Deployment’s rollout is triggered if and only if the Deployment’s pod template (that is, `.spec.template`) is changed, for example if the labels or container images of the template are updated. Other updates, such as scaling the Deployment, do not trigger a rollout.

```go
public Pod createPod() {
    int hashCode = this.getSpec().getPodTemplateSpec().hashCode();
    Pod pod = new PodBuilder()
            .withNewMetadata()
            .withLabels(this.getSpec().getLabelSelector().getMatchLabels())
            .addToLabels("pod-template-hash", String.valueOf(hashCode > 0 ? hashCode : -hashCode))
            .withName(this.getMetadata().getName()
                    .concat("-")
                    .concat(UUID.randomUUID().toString()))
            .withNamespace(this.getMetadata().getNamespace())
            .withOwnerReferences(
                    new OwnerReferenceBuilder()
                            .withApiVersion(this.getApiVersion())
                            .withController(Boolean.TRUE)
                            .withBlockOwnerDeletion(Boolean.TRUE)
                            .withKind(this.getKind())
                            .withName(this.getMetadata().getName())
                            .withUid(this.getMetadata().getUid())
                            .build()
            )
            .withUid(UUID.randomUUID().toString())
            .endMetadata()
            .withSpec(this.getSpec().getPodTemplateSpec().getSpec())
            .build();
    return pod;
}
```

## 更新MyDeployment的实现逻辑

更新时主要考虑了两种情况：缩扩容和更新镜像。

通过判断目前Pod数量和MyDeployment中spec的replicas中是否相同来判断是否需要缩扩容。

通过判断是否存在Pod与MyDeployment的container列表是否相同来判断是否需要更新镜像。

如果仅需要更新镜像，则进行rolling-update；

如果仅需要缩扩容，则进行scale；

如果都需要，则先对剩余Pod进行rolling-update，再对缩扩容的Pod进行缩扩容。

rolling-update是滚动升级，一种简单的实现是先扩容一个Pod（新的镜像），再缩容一个Pod（老的镜像），如此反复，直到全部都是新的镜像为止。

```go
public void handle(MyDeployment myDeployment) {
        log.info("{} modified", myDeployment.getMetadata().getName());
        PodList pods = delegate.client().pods().inNamespace(myDeployment.getMetadata().getNamespace()).withLabelSelector(myDeployment.getSpec().getLabelSelector()).list();
        int podSize = pods.getItems().size();
        int replicas = myDeployment.getSpec().getReplicas();
        boolean needScale = podSize != replicas;
        boolean needUpdate = pods.getItems().stream().anyMatch(pod -> {
            return myDeployment.isPodTemplateChanged(pod);
        });
        log.info("needScale: {}", needScale);
        log.info("needUpdate: {}", needUpdate);
        // 仅更新podTemplate
        if (!needScale) {
            syncRollingUpdate(myDeployment, pods.getItems());
        } else if (!needUpdate) {
            // 仅扩缩容
            int diff = replicas - podSize;
            if (diff > 0) {
                scaleUp(myDeployment, diff);
            } else {
                // 把列表前面的缩容，后面的不动
                scaleDown(pods.getItems().subList(0, -diff));
            }
        } else {
            // 同时scale&update
            // 对剩余部分做rolling-update，然后对diff进行缩扩容
            syncRollingUpdate(myDeployment, pods.getItems().subList(0, Math.min(podSize, replicas)));
            int diff = replicas - podSize;
            if (diff > 0) {
                scaleUp(myDeployment, diff);
            } else {
                scaleDown(pods.getItems().subList(replicas, podSize));
            }
        }
    }
```

值得注意的一点是，所有CRUD的APi均为REST调用，是Kubernetes API Server将对象的期望状态写入到ETCD中，然后由kubelet监听事件，去执行变更。

这一点在rolling-update过程中要格外注意，”先扩容，后缩容“中后缩容的前提是扩容成功，即新的Pod创建成功，且状态变为Ready。所以仅仅调用一下create接口是不够的，需要等待直至Ready；删除同理。

```go
private void syncRollingUpdate(MyDeployment myDeployment, List<Pod> pods) {
    pods.forEach(oldPod -> {
        Pod newPod = myDeployment.createPod();
        log.info("Thread {}: pod {} is creating", Thread.currentThread().getId(), newPod.getMetadata().getName());
        delegate.createPodAndWait(newPod, myDeployment);
        log.info("Thread {}: pod {} is deleting", Thread.currentThread().getId(), oldPod.getMetadata().getName());
        delegate.deletePodAndWait(oldPod);
    });
}
```

```go
public void createPodAndWait(Pod pod, MyDeployment myDeployment) {
    client.pods().create(pod);
    CountDownLatch latch = new CountDownLatch(1);
    modifiedPodLatchMap.put(pod.getMetadata().getUid(), latch);
    try {
        latch.await(TIME_OUT, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
       log.error("{}", e);
    }
    log.info("createPodAndWait wait finished successfully!");
}
```

这里是await阻塞等待，当前类也实现了PodModifiedWatcher接口，countDown是在Pod状态变更时触发。

当Pod被创建后，往往会触发两个事件，第一个是Added，状态是NotReady，第二个是Modified，状态是Ready。这里我们检测到新增的Pod正常运行后，去唤醒执行rolling-update的线程，以实现createPodAndwait的效果。

```go
@Override
public void onPodModified(Pod pod) {
    if (UnifiedPodWatcher.isPodReady(pod)) {
        CountDownLatch latch = modifiedPodLatchMap.remove(pod.getMetadata().getUid());
        if(latch != null) {
            latch.countDown();
        }
    }
}
```

缩扩容的逻辑比较简单，不需要做等待。

```go
/**
 * @param myDeployment
 * @param count        扩容的pod数量
 */
private void scaleUp(MyDeployment myDeployment, int count) {
    for (int i = 0; i < count; i++) {
        Pod pod = myDeployment.createPod();
        log.info("scale up pod[{}]: {} , {}", i, pod.getMetadata().getName(), pod);
        delegate.client().pods().create(pod);
    }
}

/**
 * @param pods 待删掉的pod列表
 */
private void scaleDown(List<Pod> pods) {
    for (int i = 0; i < pods.size(); i++) {
        Pod pod = pods.get(i);
        log.info("scale down pod[{}]: {} , {}", i, pod.getMetadata().getName(), pod);
        delegate.client().pods().delete(pod);
    }
}
```

## 删除MyDeployment的实现逻辑

其实就是没有逻辑=.=

Kubernetes的逻辑是删掉了一个资源时，如果其他资源的Metadata中的ownerReference中引用该资源，那么这些资源会被级联删除，这个行为是可以配置的，并且默认为true。

所以我们不需要在此处做任何事情。

# Reference
https://www.servicemesher.com/blog/kubernetes-crd-quick-start/






