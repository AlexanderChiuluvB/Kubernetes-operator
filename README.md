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
mkdir -p operator-learning && cd operator-learning
export GOPATH=$PWD 
mkdir -p $GOPATH/src/github.com/alex
cd $GOPATH/src/github.com/alex
operator-sdk new opdemo

```

2.添加api

```
operator-sdk add api --api-version=app.example.com/v1 --kind=AppService
```

3.添加控制器

```
 operator-sdk add controller --api-version=app.example.com/v1 --kind=AppService
```


#### 自定义API

打开源文件pkg/apis/app/v1/appservice_types.go,根据自定义资源所有的属性,定义相应的属性

```
import (
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    appv1 "github.com/cnych/opdemo/pkg/apis/app/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type AppServiceSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
	// Add custom validation using kubebuilder tags: https://book-v1.book.kubebuilder.io/beyond_basics/generating_crd.html
	Size  	  *int32                      `json:"size"`
	Image     string                      `json:"image"`
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
	Envs      []corev1.EnvVar             `json:"envs,omitempty"`
	Ports     []corev1.ServicePort        `json:"ports,omitempty"`
}
```

描述资源的状态

```
type AppServiceStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
	// Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
	appsv1.DeploymentStatus `json:",inline"`
}
```

定义完成后,在根目录下执行

```
operator-sdk generate k8s
```

这样就完成了对自定义资源对象的API声明


#### 实现业务逻辑

具体是在相应的controller实现的,具体在appservice_controller.go,需要去更改的核心部分是Reconcile方法,核心是不断去watch资源的状态,然后根据状态不同去实现不同的操作逻辑,核心代码如下

```
func (r *ReconcileAppService) Reconcile(request reconcile.Request) (reconcile.Result, error) {
	reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
	reqLogger.Info("Reconciling AppService")

	// Fetch the AppService instance
	instance := &appv1.AppService{}
	err := r.client.Get(context.TODO(), request.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		return reconcile.Result{}, err
	}

	if instance.DeletionTimestamp != nil {
		return reconcile.Result{}, err
	}

	// 如果不存在，则创建关联资源
	// 如果存在，判断是否需要更新
	//   如果需要更新，则直接更新
	//   如果不需要更新，则正常返回

	deploy := &appsv1.Deployment{}
	if err := r.client.Get(context.TODO(), request.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
		// 创建关联资源
		// 1. 创建 Deploy
		deploy := resources.NewDeploy(instance)
		if err := r.client.Create(context.TODO(), deploy); err != nil {
			return reconcile.Result{}, err
		}
		// 2. 创建 Service
		service := resources.NewService(instance)
		if err := r.client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 3. 关联 Annotations
		data, _ := json.Marshal(instance.Spec)
		if instance.Annotations != nil {
			instance.Annotations["spec"] = string(data)
		} else {
			instance.Annotations = map[string]string{"spec": string(data)}
		}

		if err := r.client.Update(context.TODO(), instance); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}

	oldspec := appv1.AppServiceSpec{}
	if err := json.Unmarshal([]byte(instance.Annotations["spec"]), oldspec); err != nil {
		return reconcile.Result{}, err
	}

	if !reflect.DeepEqual(instance.Spec, oldspec) {
		// 更新关联资源
		newDeploy := resources.NewDeploy(instance)
		oldDeploy := &appsv1.Deployment{}
		if err := r.client.Get(context.TODO(), request.NamespacedName, oldDeploy); err != nil {
			return reconcile.Result{}, err
		}
		oldDeploy.Spec = newDeploy.Spec
		if err := r.client.Update(context.TODO(), oldDeploy); err != nil {
			return reconcile.Result{}, err
		}

		newService := resources.NewService(instance)
		oldService := &corev1.Service{}
		if err := r.client.Get(context.TODO(), request.NamespacedName, oldService); err != nil {
			return reconcile.Result{}, err
		}
		oldService.Spec = newService.Spec
		if err := r.client.Update(context.TODO(), oldService); err != nil {
			return reconcile.Result{}, err
		}

		return reconcile.Result{}, nil
	}

	return reconcile.Result{}, nil
}

```

核心思想是先判断资源是否存在,如果不存在就直接创建新的Deployment和Service资源

另外要在resources文件夹下实现NewDeploy和NewService方法,根据CRD的声明取填充Deployment和Service对象的Spec对象


```
func NewDeploy(app *appv1.AppService) *appsv1.Deployment {
	labels := map[string]string{"app": app.Name}
	selector := &metav1.LabelSelector{MatchLabels: labels}
	return &appsv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "apps/v1",
			Kind:       "Deployment",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,

			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group: v1.SchemeGroupVersion.Group,
					Version: v1.SchemeGroupVersion.Version,
					Kind: "AppService",
				}),
			},
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: app.Spec.Size,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					Containers: newContainers(app),
				},
			},
			Selector: selector,
		},
	}
}

func newContainers(app *v1.AppService) []corev1.Container {
	containerPorts := []corev1.ContainerPort{}
	for _, svcPort := range app.Spec.Ports {
		cport := corev1.ContainerPort{}
		cport.ContainerPort = svcPort.TargetPort.IntVal
		containerPorts = append(containerPorts, cport)
	}
	return []corev1.Container{
		{
			Name: app.Name,
			Image: app.Spec.Image,
			Resources: app.Spec.Resources,
			Ports: containerPorts,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Env: app.Spec.Envs,
		},
	}
}

```

```
func NewService(app *v1.AppService) *corev1.Service {
	return &corev1.Service {
		TypeMeta: metav1.TypeMeta {
			Kind: "Service",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name: app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group: v1.SchemeGroupVersion.Group,
					Version: v1.SchemeGroupVersion.Version,
					Kind: "AppService",
				}),
			},
		},
		Spec: corev1.ServiceSpec{
			Type: corev1.ServiceTypeNodePort,
			Ports: app.Spec.Ports,
			Selector: map[string]string{
				"app": app.Name,
			},
		},
	}
}
```

在集群中安装CRD对象

```
kubectl create -f deploy/crds/app_v1_appservice_crd.yaml
```

然后在本地项目中启动Operator进行调试

```
operator-sdk up local 
```

只要成功创建了crd,就可以添加这个自定义资源类型的资源了

例如crd的名称是AppService,那么我们用`kubectl create -f cr.yaml` 来创建这个资源

```
apiVersion: app.example.com/v1
kind: AppService
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

创建完成后,就可以观察到其附带创建了service,deployment,pod这些关联的资源

### Ref
[K8s入门教程](https://www.qikqiak.com/post/k8s-operator-101/)
