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

### Demo

#### 如果没有Operator

如果你要部署一个WebServer到K8s集群中,需要

1.编写一个Deployment,描述Pod的个数,状态

2.创建Service,提供服务,并且通过Pod的Label进行关联

3.暴露服务:Ingress或者是Node-port类型的Service

#### 如果有Operator

我们只需要写一个CRD,把Pod的镜像,服务暴露的端口,环境变量都写下来

简而言之,CRD可以包揽了创建Deployment和Service的任务,一个yaml文件完成了Deployment和Service所要做的事情

举个例子:

```
apiVersion: app.example.com/v1
kind: WebService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```

上面定义了一个名为"WebService"的CRD资源,取创建副本数为2的Nginx Pod,然后通过nodePort=30002向外提供服务


### 开发环境

1.Kubernetes集群 (我这里用的是minikube)

2.golang 不多说

3.golang的dep工具包

```
sudo apt-get install go-dep
```
4.安装[operator-sdk](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md)


### 开发流程

1.
```
export GOPATH=$PWD 
mkdir -p $GOPATH/src/github.com/alex
cd $GOPATH/src/github.com/alex
operator-sdk new opdemo

```

2.



### Ref
[K8s入门教程](https://www.qikqiak.com/post/k8s-operator-101/)
