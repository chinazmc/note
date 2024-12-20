#k8s #pod

Pod由一组应用容器组成，其中包含共有的环境和资源约束。
在Kubernetes系统中对长时间运行容器的要求是：其主程序需要一直在前台执行。

对于无法改造为前台执行的应用，也可以使用开源工具Supervisor辅助进行前台运行的功能。Supervisor提供了一种可以同时启动多个后台应用，并保持Supervisor自身在前台执行的机制，可以满足Kubernetes对容器的启动要求。关于Supervisor的安装和使用，请参考官网http://supervisord.org的文档说明。

静态Pod是由kubelet进行管理的仅存在于特定Node上的Pod。它们不能通过APIServer进行管理，无法与ReplicationController、Deployment或者DaemonSet进行关联，并且kubelet无法对它们进行健康检查。静态Pod总是由kubelet创建的，并且总在kubelet所在的Node上运行。
由于静态Pod无法通过API Server直接管理，所以在Master上尝试删除这个Pod时，会使其变成Pending状态，且不会被删除。
删除该Pod的操作只能是到其所在Node上将其定义文件static-web.yaml从/etc/kubelet.d目录下删除。
应用部署的一个最佳实践是将应用所需的配置信息与程序进行分离，这样可以使应用程序被更好地复用，通过不同的配置也能实现更灵活的功能。

ConfigMap供容器使用的典型用法如下。
（1）生成为容器内的环境变量。
（2）设置容器启动命令的启动参数（需设置为环境变量）。
（3）以Volume的形式挂载为容器内部的文件或目录。


3.5.4 使用ConfigMap的限制条件使用ConfigMap的限制条件如下。
◎ ConfigMap必须在Pod之前创建。
◎ ConfigMap受Namespace限制，只有处于相同Namespace中的Pod才可以引用它。
◎ ConfigMap中的配额管理还未能实现。
◎ kubelet只支持可以被API Server管理的Pod使用ConfigMap。kubelet在本Node上通过--manifest-url或--config自动创建的静态Pod将无法引用ConfigMap。
◎ 在Pod对ConfigMap进行挂载（volumeMount）操作时，在容器内部只能挂载为“目录”，无法挂载为“文件”。在挂载到容器内部后，在目录下将包含ConfigMap定义的每个item，如果在该目录下原来还有其他文件，则容器内的该目录将被挂载的ConfigMap覆盖。如果应用程序需要保留原来的其他文件，则需要进行额外的处理。可以将ConfigMap挂载到容器内部的临时目录，再通过启动脚本将配置文件复制或者链接到（cp或link命令）应用所用的实际配置目录下。

