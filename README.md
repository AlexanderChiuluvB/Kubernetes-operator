# Kubernetes-operator
Tutorial of how to write a k8s operator


### What is K8s-operator

Operator是用于扩展K8s API的应用程序控制器,用于创建,配置和管理负责的**有状态应用**

那什么是有状态应用呢?

例如数据库,缓存和监控系统(普罗米修斯)

创建一个K8s-operator,关键是CRD的设计

### What is CRD?

Custom Resource Definition,自定义资源的声明

K8s 1.7引入了Custom Resource的概念,可以让开发人员自定义编写资源,而不必使用K8s原生的资源(如Deployment,Replicaset,Statefulset)等来进行应用的管理.Operator直接使用K8s API来进行开发,可以**根据自定义规则**来监控集群,对应用扩缩容,更改Pod和Service等

### Operator Framework

* Operator SDK:屏蔽K8s API底层,让开发者能够自己构建一个Operator
* Operator Lifecycle Manager OLM: 帮助你安装、更新和管理跨集群的运行中的所有 Operator（以及他们的相关服务

### Workflow

1.使用SDK创建一个新的Operator项目
2.添加CRD来定义新的自定义资源API
3.使用SDK API来监控资源
4.Operator的协调逻辑
5.Operator SDK构建并生成Operator部署清单文件




### Ref
[K8s入门教程](https://www.qikqiak.com/post/k8s-operator-101/)
