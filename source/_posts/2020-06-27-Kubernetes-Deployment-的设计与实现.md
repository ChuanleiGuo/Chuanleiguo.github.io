---
title: Kubernetes Deployment 的设计与实现
date: 2020-06-27 11:12:17
categories:
- 计算机科学
mathjax: true
tags: 
- "kubernetes"
- "deployment"
- "controller"
keywords:
- "kubernetes"
- "deployment"
- "controller"
- "pod"
---

## 概念

对于在生产或测试环境使用过 Kubernetes,或者对 Kubernetes 有基本了解的用户，Deployment 一定不陌生。Deployment 为 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 和 [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 提供了声明式管理的功能，每一个 Deployment 对应集群中的一次部署。用户可以利用 Yaml 或 JSON 文件描述的期望状态，Deployment Controller 将会以一定的速度将实际状态调整为与期望状态相同。

本文主要介绍 Deployment 的工作原理，并从源码角度分析它如何实现水平扩缩容、回滚、滚动更新等功能。

## 工作原理

Deployment 是 Kubernetes 中最常见，也是最常用的 API 对象。上一篇文章中，介绍了 Kubernetes 的重要编程模型：控制器模型。Deployment Controller 即通过控制器模型为 Deployment 对象实现了声明式 API。

Deployment 看似简单，但是它实现了 Kubernetes 中非常重要的功能呢：Pod 的“水平扩展/收缩”（Horizontal scaling out/in）。而这个功能是从 PaaS 时代开始，任何一个平台级项目都必须具备的编排能力。而这个能力的实现，又依赖 Kubernetes 中的另一个重要的 API 对象：ReplicaSet。ReplicaSet 的 Yaml 非常简单，包含一个对象元数据、副本数定义和一个 Pod 模板:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable
```

不难发现 ReplicaSet 实际上是 Deployment 的一个子集。实际上，Deployment 直接管理的并不是 Pod，而是 ReplicaSet 对象。所以，对于一个 Deployment 所管理的 Pod，它的 `ownerReference` 其实是 ReplicaSet。

以一个副本数为 4 的 Deployment 为例：

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
```

在实际上，Deployment、ReplicaSet、Pod 之间的关系是：

{% asset_image dep-rs-pod.png %}

由上图可以看出，副本数是 4 的 Deployment，与 ReplicaSet，以及 Pod 是一种层层控制关系。其中 ReplicaSet 通过控制器，保证集群中 Pod 的个数永远等于指定的个数。在此基础上，Deployment 同样通过控制器模式，来操作 ReplicaSet 的个数和属性，进而实现“水平拓展/收缩”及“滚动更新”两个编排动作。其中“水平拓展/收缩”较为简单，Deployment Controller 只要修改它控制的 ReplicaSet 的副本数，ReplicaSet Controller 则根据期望的副本数新建或删除 Pod。

下面通过一个例子来解释“滚动更新”。首先，我们创建一个 `nginx-deployment`：

```yaml
kubectl create -f nginx-deploy.yaml --record
```

然后，检查创建后的状态：

```yaml

$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4         0         0            0           1s
```

1. DESIRED：表示期望的 Pod 个数；
2. CURRENT：当前 Running 状态 Pod 个数；
3. UP-TO-DATE：处于最新版本 Pod 个数；
4. AVAILABLE：当前已经可用的 Pod 个数，即既是 Running，又是最新版本，并且健康检查已为 Ready 的 Pod 个数；

待全部 Pod 均处于 AVAILABLE 状态后，可以再查询 Deployment 所控制的 ReplicaSet：

```yaml

$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-3167673210   4         4         4       20s
```

可以看到，创建 Deployment 后，Deployment Controller 就会立即创建一个 Pod 副本数为 4 的 ReplicaSet。而 ReplicaSet 的名字，则是由 Deployment 的名字和一个随机字符串组成。Deployment 的状态是在 ReplicaSet 的基础上，添加了和版本相关的 `UP-TO-DATE` 字段。

这时候，如果修改 Deployment 的 Pod 模板，就会触发“滚动更新”。当使用 `kubectl edit deployment` 或 `kubectl apply` 提交新的 yaml 后，可以通过查看 Deployment 的 Events，看到这个“滚动更新”的流程：

```bash

$ kubectl describe deployment nginx-deployment
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
...
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 1
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 2
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 2
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 1
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 3
  Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 0
```

可以看到当修改 PodTemplate 后，Deployment Controller 会使用这个新的 PodTemplate 创建一个新的 ReplicaSet，它初始的副本数是 0。然后 Deployment Controller 开始将新的 ReplicaSet 所控制的 Pod 副本数由 0 个变成 1 个，即：“水平拓展”出一个副本；接着又将旧的 ReplicaSet 所控制的 Pod 副本数减少一个，即：“水平收缩”。如此交替进行，最后新的 ReplicaSet 所管理的 Pod 个数上升为 4，而旧的 ReplicaSet 所管理的 Pod 个数收缩为 0 个。

这样，将集群中正在运行的多个 Pod 版本，交替逐一升级更换的过程就是“滚动更新”。更新结束后，可以查看新、旧两个 ReplicaSet 的状态：

```bash

$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1764197365   4         4         4       6s
nginx-deployment-3167673210   0         0         0       30s
```

滚动更新有很明显的好处。比如当新的 Pod 因为故障无法启动时，“滚动更新”就会停止，允许开发者介入。而此时应用本身还是有两个旧版本的 Pod 在线，所以服务不会受到太大的影响。当然这也要求开发者一定要使用 Health Check 检查应用的运行状态，而不是依赖于容器的 Running 状态。此外，Deployment Controller 还会保证在任何时间窗口内，只有指定比例的新 Pod 被创建，及指定比例的旧 Pod 处于离线状态。这两个比例的默认值均为 25%，且可以在 `spec.rollingUpdateStrategy` 中配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

结合“滚动更新”，Deployment、ReplicaSet、Pod 的关系图为：

{% asset_image dep-rs-pod-2.png %}

如图中所示，Deployment Controller 实际上控制 ReplicaSet 的数量，以及每个 ReplicaSet 的属性。而应用的一个版本对应的正是一个 ReplicaSet，这个版本的 Pod 数量则由 ReplicaSet Controller 保证。通过多个 ReplicaSet，Kubernetes 就实现了对多个“应用版本”的描述。

当在滚动更新，发现新版本不符合预期，需要回滚到旧版本的时候，可以执行:

```bash

$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment
```

在具体操作上，Deployment Controller 就是让旧的 ReplicaSet 再次扩展成 4 个 Pod，而让新的 Pod 重新收缩为 0 个。那么更进一步，如果需要回滚到更早的版本，要怎么办呢？

首先，需要使用 `kubectl roll history` 命令，查看每次 Deployment 变更对应的版本：

```yaml

$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:alpine
```

然后可以在回滚的时候通过指定版本的版本号，回滚到指定版本：

```yaml

$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.extensions/nginx-deployment
```

这里我们可能已经想到一个问题：对 Deployment 的每次更新操作，都会产生一个新的 ReplicaSet，这会不会浪费资源呢？

其中一个方法是可以使用 `kubectl rollout pause` 指令暂停滚动更新，当完成对 Deployment 的全部修改后，在使用 `kubectl rollout resume` 指令恢复继续进行滚动更新。这时，只会生成一个 ReplicaSet。不过，其实这样控制 ReplicaSet 的数量，随着应用的不断更新，ReplicaSet 的数量还是会不断增加。Deployment 利用一个 `spec.revisionHistoryLimit` 字段，限制了保存的历史版本个数。所以如果将它设置为 0，那么就无法进行回滚操作了。

## 源码分析

前文我们从实践角度分析了 Deployment 的工作原理。下面通过分析源码，研究 Deployment 的具体实现。

在 Kubernetes 的结构中存在一个叫做 `kube-controller-manager` 的组件。其实这个组件就是一系列控制器的组合。在项目源代码 `pkg/controller` 目录下可以看到一系列 controller：

```yaml

$ cd kubernetes/pkg/controller/
$ ls -d */              
deployment/             job/                    podautoscaler/          
cloud/                  disruption/             namespace/              
replicaset/             serviceaccount/         volume/
cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
...
```

这个目录下的每一个控制器，都以独特的方式负责某种编排功能，都遵循通用的编排功能：控制循环。而 Deployment 正式控制器中的一种。

`DeploymentController` 作为管理 Deployment 资源的控制器，会在启动时利用 `Informer` 监听 Pod、ReplicaSet、Deployment 三种资源对象的通知，这三种资源的变动都会触发控制器中的回调函数。

不同对象的事件在被过滤后进行工作队列，等待工作进程的消费，以下事件会触发 Deployment 的同步：

1. Deployment 变化；
2. Deployment 控制的 ReplicaSet 的变化；
3. Deployment 下的 Pod 数量为 0 时，Pod 的删除事件；

```go
// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, 
                             rsInformer appsinformers.ReplicaSetInformer,         
                             podInformer coreinformers.PodInformer, 
                             client clientset.Interface) 
                             (*DeploymentController, error) {
    
    dc := &DeploymentController{
		client:        client,
		eventRecorder: eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
		queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
    }
    
    dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
		DeleteFunc: dc.deleteDeployment,
	})
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: dc.deletePod,
    })
    ...
	return dc, nil
}
```

DeploymentController 在调用 Run 的时候启动多个工作进程，这些工作进程运行 worker 方法从队列中读取最新 Deployment 对象进行同步。

### 同步

DeploymentController 对 Deployment 的通过通过以 `syncDeployment` 方法进行，该方法包含同步、回滚及更新逻辑，也是同步 Deployment 资源的唯一入口：

```go
// syncDeployment will sync the deployment with the given key.
// This function is not meant to be invoked concurrently with the same key.
func (dc *DeploymentController) syncDeployment(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(2).Infof("Deployment %v has been deleted", key)
		return nil
	}
	if err != nil {
		return err
    }
    
    d := deployment.DeepCopy()
    // List ReplicaSets owned by this Deployment, while reconciling ControllerRef
	// through adoption/orphaning.
    rsList, _ := dc.getReplicaSetsForDeployment(d)
    // List all Pods owned by this Deployment, grouped by their ReplicaSet.
	// Current uses of the podMap are:
	//
	// * check if a Pod is labeled correctly with the pod-template-hash label.
	// * check that no old Pods are running in the middle of Recreate Deployments.
    podMap, _ := dc.getPodMapForDeployment(d, rsList)
    
    if d.Spec.Paused {
		return dc.sync(d, rsList)
    }
    
    scalingEvent, err := dc.isScalingEvent(d, rsList)
	if err != nil {
		return err
	}
	if scalingEvent {
		return dc.sync(d, rsList)
    }
    
    switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		return dc.rolloutRolling(d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
```

在删除简化后的流程中：

1. 由传入的 key 获取 Deployment；
2. 调用 `getReplicaSetsForDeployment` 获取集群中与 Deployment 相关的所有 ReplicaSet；
    
    1. 查找所有 ReplicaSet；
    2. 根据 Deployment 中的选择器对 ReplicaSet 建立或释放从属关系；

3. 调用 `getPodMapForDeployment` 查询 Deployment 控制的 ReplicaSet 到 Pod 的映射；

    1. 根据选择器查询全部 Pod；
    2. 根据 Pod 的控制器 ReplicaSet 对上述 Pod 进行分类；

4. 如果 Deployment 处于暂停状态或者需要扩容，就会调用 `sync` 方法同步 Deployment；
5. 在正常情况下会根据规格中的策略对 Deployment 进行更新：

    1. `Recreate` 策略调用 `rolloutRecreate` 方法，先杀掉所有存在的 Pod 后启动新的 Pod 副本；
    2. `RollingUpdate` 策略调用 `rolloutRolling` 方法，根据 `maxSurge` 和 `maxUnavailable` 配置对 Pod 进行滚动更新。


### 扩容

如果当前需要更新的 Deployment 由 `isScalingEvent` 检查发现更新事件是一次扩缩容事件，那么 ReplicaSet 持有的 Pod 数量和规格中的 ReplicaSet 不一致，那么就调用 `sync` 方法对 Deployment 进行同步：

```go
// sync is responsible for reconciling deployments on scaling events or when they
// are paused.
func (dc *DeploymentController) sync(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
    newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
	if err != nil {
		return err
	}
	if err := dc.scale(d, newRS, oldRSs); err != nil {
		// If we get an error while trying to scale, the deployment will be requeued
		// so we can abort this resync
		return err
    }
    
    allRSs := append(oldRSs, newRS)
	return dc.syncDeploymentStatus(allRSs, newRS, d)
}
```

`sync` 从 apiserver 获取当前 Deployment 对应的最新 ReplicaSet 和历史 ReplicaSet 并调用 `scale` 方法开始扩容。`scale` 即为扩容主要执行的方法。该方法较长，本文分为几部分来分析具体实现：

```go
func (dc *DeploymentController) scale(deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
    // If there is only one active replica set then we should scale that up to the full count of the
	// deployment. If there is no active replica set, then we should scale up the newest replica set.
	if activeOrLatest := deploymentutil.FindActiveOrLatest(newRS, oldRSs); activeOrLatest != nil {
		if *(activeOrLatest.Spec.Replicas) == *(deployment.Spec.Replicas) {
			return nil
		}
		_, _, err := dc.scaleReplicaSetAndRecordEvent(activeOrLatest, *(deployment.Spec.Replicas), deployment)
		return err
	}

	// If the new replica set is saturated, old replica sets should be fully scaled down.
	// This case handles replica set adoption during a saturated new replica set.
	if deploymentutil.IsSaturated(deployment, newRS) {
		for _, old := range controller.FilterActiveReplicaSets(oldRSs) {
			if _, _, err := dc.scaleReplicaSetAndRecordEvent(old, 0, deployment); err != nil {
				return err
			}
		}
		return nil
	}
}
```

假如集群中只有一个活跃的 ReplicaSet，那么就对该 ReplicaSet 进行直接扩缩容，但是如果不存在活跃 ReplicaSet 则选择最新 ReplicaSet。这部分工作由 `FindActiveOrLatest` 及 `scaleReplicaSetAndRecordEvent` 共同完成。

当调用 `IsSaturated` 发现当前 Deployment 对应的副本数已饱和时，则删除所有历史版本 ReplicaSet 持有的 Pod 副本。

当 Deployment 使用滚动更新策略时，如果发现当前 ReplicaSet 没有饱和并且存在多个活跃的 ReplicaSet 对象，则按照比例分别对各个活跃的 ReplicaSet 进行扩容或缩容：

```go
    // There are old replica sets with pods and the new replica set is not saturated.
	// We need to proportionally scale all replica sets (new and old) in case of a
	// rolling deployment.
	if deploymentutil.IsRollingUpdate(deployment) {
		allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))
		allRSsReplicas := deploymentutil.GetReplicaCountForReplicaSets(allRSs)

		allowedSize := int32(0)
		if *(deployment.Spec.Replicas) > 0 {
			allowedSize = *(deployment.Spec.Replicas) + deploymentutil.MaxSurge(*deployment)
		}

		// Number of additional replicas that can be either added or removed from the total
		// replicas count. These replicas should be distributed proportionally to the active
		// replica sets.
		deploymentReplicasToAdd := allowedSize - allRSsReplicas

		// The additional replicas should be distributed proportionally amongst the active
		// replica sets from the larger to the smaller in size replica set. Scaling direction
		// drives what happens in case we are trying to scale replica sets of the same size.
		// In such a case when scaling up, we should scale up newer replica sets first, and
		// when scaling down, we should scale down older replica sets first.
		var scalingOperation string
		switch {
		case deploymentReplicasToAdd > 0:
			sort.Sort(controller.ReplicaSetsBySizeNewer(allRSs))
			scalingOperation = "up"

		case deploymentReplicasToAdd < 0:
			sort.Sort(controller.ReplicaSetsBySizeOlder(allRSs))
			scalingOperation = "down"
		}
```

1. 调用 `FilterActiveReplicaSets` 查询所有活跃的 ReplicaSet；
2. 调用 `GetReplicaCountForReplicaSets` 计算当前 Deployment 对应 ReplicaSet 持有的全部 Pod 副本个数；
3. 根据 Deployment 配置的 `maxSurge` 和 `replicas` 计算允许创建的 Pod 数量；
4. 利用 `allowedSize` 和 `allRSsReplicas` 计算出需要增加或者删除的副本数；
5. 根据 `deploymentReplicasToAdd` 变量的符号对 ReplicaSet 数组进行排序并确定当前的操作是扩容还是缩容：

    1. 若 `deploymentReplicasToAdd` > 0，ReplicaSet 按照从新到旧的顺序进行扩容；
    2. 若 `deploymentReplicasToAdd` < 0，ReplicaSet 按照从旧到新的顺序进行缩容；

```go
        // Iterate over all active replica sets and estimate proportions for each of them.
		// The absolute value of deploymentReplicasAdded should never exceed the absolute
		// value of deploymentReplicasToAdd.
		deploymentReplicasAdded := int32(0)
		nameToSize := make(map[string]int32)
		for i := range allRSs {
			rs := allRSs[i]

			// Estimate proportions if we have replicas to add, otherwise simply populate
			// nameToSize with the current sizes for each replica set.
			if deploymentReplicasToAdd != 0 {
				proportion := deploymentutil.GetProportion(rs, *deployment, deploymentReplicasToAdd, deploymentReplicasAdded)

				nameToSize[rs.Name] = *(rs.Spec.Replicas) + proportion
				deploymentReplicasAdded += proportion
			} else {
				nameToSize[rs.Name] = *(rs.Spec.Replicas)
			}
		}
```

由于当前的 Deployment 持有了多个活跃的 ReplicaSet，所以在计算了需要增加或者删除的副本数 `deploymentReplicasToAdd`后，就会为多个活跃的 ReplicaSet 分配需要改变的副本数，`GetProportion` 会根据以下几个参数确定结果:

1. Deployment 期望的 Pod 副本数量；
2. 需要新增或减少的副本数量；
3. Deployment 目前通过 ReplicaSet 持有的 Pod 总数；

Kubernetes 在 `getReplicaSetFraction` 中使用下面的公式计算每个 ReplicaSet 在 Deployment 资源中的占比，最后返回该 ReplicaSet 需要改变的副本数：

$$
replicaSet.Spec.Replicas \times \left( \frac{deployment.Spec.Replicas + maxSurge(deployment)}{deployment.Status.Replicas} - 1 \right)
$$

该结果又会与目前期望的剩余变化量对比，保证变化的副本数量不会超过期望值。

```go
		// Update all replica sets
		for i := range allRSs {
			rs := allRSs[i]

			// Add/remove any leftovers to the largest replica set.
			if i == 0 && deploymentReplicasToAdd != 0 {
				leftover := deploymentReplicasToAdd - deploymentReplicasAdded
				nameToSize[rs.Name] = nameToSize[rs.Name] + leftover
				if nameToSize[rs.Name] < 0 {
					nameToSize[rs.Name] = 0
				}
			}

			// TODO: Use transactions when we have them.
			if _, _, err := dc.scaleReplicaSet(rs, nameToSize[rs.Name], deployment, scalingOperation); err != nil {
				// Return as soon as we fail, the deployment is requeued
				return err
			}
		}
```

`scale` 方法的最后会直接调用 `scaleReplicaSet` 将每一个 ReplicaSet 都扩容或缩容到期望的副本数：

```go
func (dc *DeploymentController) scaleReplicaSet(rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment, scalingOperation string) (bool, *apps.ReplicaSet, error) {

	sizeNeedsUpdate := *(rs.Spec.Replicas) != newScale

	annotationsNeedUpdate := deploymentutil.ReplicasAnnotationsNeedUpdate(rs, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))

	scaled := false
	var err error
	if sizeNeedsUpdate || annotationsNeedUpdate {
		rsCopy := rs.DeepCopy()
		*(rsCopy.Spec.Replicas) = newScale
		deploymentutil.SetReplicasAnnotations(rsCopy, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))
		rs, err = dc.client.AppsV1().ReplicaSets(rsCopy.Namespace).Update(context.TODO(), rsCopy, metav1.UpdateOptions{})
		if err == nil && sizeNeedsUpdate {
			scaled = true
			dc.eventRecorder.Eventf(deployment, v1.EventTypeNormal, "ScalingReplicaSet", "Scaled %s replica set %s to %d", scalingOperation, rs.Name, newScale)
		}
	}
	return scaled, rs, err
}
```

该方法会直接修改目标 ReplicaSet.Spec 中的 Replicas 参数和注解 `deployment.kubernetes.io/desired-replicas` 的值并通过 API 请求更新当前的 ReplicaSet 对象。

用户可以通过 `kubectl describe` 命令查看 ReplicaSet 的 Annotations，其实能发现当前 RS 的期待副本数和最大副本数：

```bash
$ kubectl describe rs nginx-deployment-76bf4969df
Name:           nginx-deployment-76bf4969df
Namespace:      default
Selector:       app=nginx,pod-template-hash=76bf4969df
Labels:         app=nginx
                pod-template-hash=76bf4969df
Annotations:    deployment.kubernetes.io/desired-replicas=4
                deployment.kubernetes.io/max-replicas=5
...
```

### 重新创建

当 Deployment 使用的更新策略是 `Recreate` 时，DeploymentController 就会使用如下的 `rolloutRecreate` 方法对 Deployment 进行更新：

```go
// rolloutRecreate implements the logic for recreating a replica set.
func (dc *DeploymentController) rolloutRecreate(d *apps.Deployment, rsList []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) error {
	// Don't create a new RS if not already existed, so that we avoid scaling up before scaling down.
	newRS, oldRSs, _ := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
	allRSs := append(oldRSs, newRS)
	activeOldRSs := controller.FilterActiveReplicaSets(oldRSs)

	// scale down old replica sets.
	scaledDown, _ := dc.scaleDownOldReplicaSetsForRecreate(activeOldRSs, d)
	if scaledDown {
		// Update DeploymentStatus.
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// Do not process a deployment when it has old pods running.
	if oldPodsRunning(newRS, oldRSs, podMap) {
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// If we need to create a new RS, create it now.
	if newRS == nil {
		newRS, oldRSs, _ = dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
		allRSs = append(oldRSs, newRS)
	}

	// scale up new replica set.
	if _, err := dc.scaleUpNewReplicaSetForRecreate(newRS, d); err != nil {
		return err
	}

	if util.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(oldRSs, d); err != nil {
			return err
		}
	}

	// Sync deployment status.
	return dc.syncRolloutStatus(allRSs, newRS, d)
}
```

1. 调用 `getAllReplicaSetsAndSyncRevision` 和 `FilterActiveReplicaSets` 两个方法获取 Deployment 中所有的 ReplicaSet 以及其中活跃的 ReplicaSet 对象；
2. 调用 `scaleDownOldReplicaSetsForRecreate` 方法将所有活跃的历史 ReplicaSet 持有的 Pod 数降低至 0；
3. 同步 Deployment 的最新状态并等待 Pod 的终止；
4. 在需要时通过 `getAllReplicaSetsAndSyncRevision` 方法创建新的 ReplicaSet 并调用 `scaleUpNewReplicaSetForRecreate` 函数对 ReplicaSet 进行扩容；
5. 更新完成之后会调用 cleanupDeployment 方法删除历史全部的 ReplicaSet 对象并更新 Deployment 的状态；

也就是说在更新的过程中，之前创建的 ReplicaSet 和 Pod 资源会被全部删除，只是 Pod 会先被删除而 ReplicaSet 会后被删除；上述方法也会创建新的 ReplicaSet 和 Pod 对象。但是需要注意旧的 Pod 副本一定会被先删除，所以会有一段时间不存在可用的 Pod。

### 滚动更新

在使用 Deployment 对象时，我们更常用的更新策略是 `RollingUpdate`。在介绍滚动更新流程前，需要先了解两个参数：

1. `maxUnavailable`：表示在更新过程中能够进入不可用状态的 Pod 数量的最大值；
2. `maxSurge`：表示在更新过程汇总能够额外创建的 Pod 数量的最大值；

`maxUnavailable` 和 `maxSurge` 这两个滚动更新所使用的配置都可以用百分比或者绝对值表示；当使用百分比时，会使用 $Replicas \times Strategy.RollingUpdate.MaxSurge$ 计算得到相应的值。

`rolloutRolling` 为处理滚动更新的方法：

```go
// rolloutRolling implements the logic for rolling a new replica set.
func (dc *DeploymentController) rolloutRolling(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
	if err != nil {
		return err
	}
	allRSs := append(oldRSs, newRS)

	// Scale up, if we can.
	scaledUp, err := dc.reconcileNewReplicaSet(allRSs, newRS, d)
	if err != nil {
		return err
	}
	if scaledUp {
		// Update DeploymentStatus
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// Scale down, if we can.
	scaledDown, err := dc.reconcileOldReplicaSets(allRSs, controller.FilterActiveReplicaSets(oldRSs), newRS, d)
	if err != nil {
		return err
	}
	if scaledDown {
		// Update DeploymentStatus
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	if deploymentutil.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(oldRSs, d); err != nil {
			return err
		}
	}

	// Sync deployment status
	return dc.syncRolloutStatus(allRSs, newRS, d)
}
```

1. 首先，获取 Deployment 持有的全部 ReplicaSet 的资源；
2. 调用 `reconcileNewReplicaSet` 调节新的 ReplicaSet 的副本数，创建新的 Pod 并保证额外的副本数量不超过 `maxSurge`;
3. 调用 `reconcileOldReplicaSets` 调节历史 ReplicaSet 的副本数，删除旧的 Pod 并保证不可用的部分不超过 `maxUnavailable`;
4. 删除无用的 ReplicaSet 并更新 Deployment 的状态；

注意，在滚动更新过程中，Kubernetes 不是一次性就切换到期望的状态，即「目标副本数」，而是先启动新的 ReplicaSet 及一部分 Pod，然后删除历史 ReplicaSet 中的部分；如此往复，最终达到集群期望的状态。

当使用 `reconcileNewReplicaSet` 对新 ReplicaSet 进行调节时，如果发现新 ReplicaSet 中副本数满足期望则直接返回，在超过期望时则缩容。：

```go
func (dc *DeploymentController) reconcileNewReplicaSet(allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
	if *(newRS.Spec.Replicas) == *(deployment.Spec.Replicas) {
		// Scaling not required.
		return false, nil
	}
	if *(newRS.Spec.Replicas) > *(deployment.Spec.Replicas) {
		// Scale down.
		scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, *(deployment.Spec.Replicas), deployment)
		return scaled, err
	}
	newReplicasCount, err := deploymentutil.NewRSNewReplicas(deployment, allRSs, newRS)
	if err != nil {
		return false, err
	}
	scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, newReplicasCount, deployment)
	return scaled, err
}
```

如果 ReplicaSet 的数量不够则调用 `NewRSNewReplicas` 计算新的副本个数，计算过程为：

```go
currentPodCount := GetReplicaCountForReplicaSets(allRSs)
maxTotalPods := *(deployment.Spec.Replicas) + int32(maxSurge)
// Scale up.
scaleUpCount := maxTotalPods - currentPodCount
// Do not exceed the number of desired replicas.
scaleUpCount = Min(int(scaleUpCount), int(*(deployment.Spec.Replicas)-*(newRS.Spec.Replicas))))
return *(newRS.Spec.Replicas) + scaleUpCount
```

该过程中需要考虑 Deployment 期望的副本数、当前可用的副本数记忆新的 RS 持有的副本数，此外还有最大最小值的限制。

另一个滚动更新中使用的方法 `reconcileOldReplicaSets` 主要作用是对历史 ReplicaSet 对象持有的副本数量进行缩容：

```go
func (dc *DeploymentController) reconcileOldReplicaSets(allRSs []*apps.ReplicaSet, oldRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
	oldPodsCount := deploymentutil.GetReplicaCountForReplicaSets(oldRSs)
	if oldPodsCount == 0 {
		// Can't scale down further
		return false, nil
	}

	allPodsCount := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
	klog.V(4).Infof("New replica set %s/%s has %d available pods.", newRS.Namespace, newRS.Name, newRS.Status.AvailableReplicas)
	maxUnavailable := deploymentutil.MaxUnavailable(*deployment)
	minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
	newRSUnavailablePodCount := *(newRS.Spec.Replicas) - newRS.Status.AvailableReplicas
	maxScaledDown := allPodsCount - minAvailable - newRSUnavailablePodCount
	if maxScaledDown <= 0 {
		return false, nil
	}
	oldRSs, cleanupCount, err := dc.cleanupUnhealthyReplicas(oldRSs, deployment, maxScaledDown)
	if err != nil {
		return false, nil
	}
	klog.V(4).Infof("Cleaned up unhealthy replicas from old RSes by %d", cleanupCount)

	// Scale down old replica sets, need check maxUnavailable to ensure we can scale down
	allRSs = append(oldRSs, newRS)
	scaledDownCount, err := dc.scaleDownOldReplicaSetsForRollingUpdate(allRSs, oldRSs, deployment)
	if err != nil {
		return false, nil
	}
	klog.V(4).Infof("Scaled down old RSes of deployment %s by %d", deployment.Name, scaledDownCount)

	totalScaledDown := cleanupCount + scaledDownCount
	return totalScaledDown > 0, nil
}
```

1. 计算历史 ReplicaSet 持有的副本总数；
2. 计算全部 ReplicaSet 持有的副本总数；
3. 根据 Deployment 期望的副本数、最大不可用的副本数以及新的 ReplicaSet 中不可用的 Pod 数量计算最大缩容个副本个数；
4. 利用 `cleanupUnhealthyReplicas` 清理 ReplicaSet 中处于不健康状态的副本；
5. 利用 `scaleDownOldReplicaSetsForRollingUpdate` 对历史 ReplicaSet 中的副本进行缩容；

### 回滚

Kubernetes 中的每一个 Deployment 资源都包含 `revision` 概念，版本的使用可以让我们在更新不符合预期是及时通过 Deployment 的版本对其进行回滚。当我们更新 Deployment 时，之前 Deployment 持有的 ReplicaSet 会被清理。Deployment 通过规格中的 `revisionHistoryLimit` 字段配置最多保留的 ReplicaSet 数量，及多少个版本，这些 ReplicaSet 并不会被删除，它们只是不持有任何的 Pod 副本。

保留这些资源能够方便 Deployment 进行回滚，回滚荣国客户端调用 `rollout undo` 命令实现：

```go
kubectl rollout undo deployment.v1.apps/nginx-deployment
deployment.apps/nginx-deployment
```

上述命令没有指定版本号，所以默认回滚到上一个版本。如果在回滚时指定版本，那么 Kubernetes 就会根据传入的版本查找历史的 ReplicaSet 资源，并触发一个资源更新请求.

回滚对于 Kubernetes 来说与更新操作没有区别，在每次更新时都会根据模板在历史 ReplicaSet 中查询是否有相同的 ReplicaSet 存在。如果存在规格完全相同的 ReplicaSet，就会保留这个 ReplicaSet 历史上使用的版本号并对该 ReplicaSet 重新扩容并对正在工作的 ReplicaSet 进行缩容以实现期望状态。

### 删除

如果用户在 Kubernetes 中删除了一个 Deployment 资源，那么 Deployment 持有的 ReplicaSet 以及 ReplicaSet 持有的副本都会被 Kubernetes 中的垃圾收集器删除。

由于和当前 Deployment 有关的 ReplicaSet 历史和最新版本都会被删除，所以对应的 Pod 副本也都会随之被删除，这些字段都是通过 `metadata.ownerReference` 字段关联。

## 总结

本文分析了 Deployment 这个在 Kubernetes 中最常使用的编排控制的实现和工作原理。

Deployment 实际上是一个两层控制器。首先它通过 ReplicaSet 的个数描述应用的版本；然后通过 ReplicaSet 的属性，保证 Pod 的数量。

> Deployment 控制 ReplicaSet（版本），ReplicaSet 控制 Pod（副本数）。

Deployment 的设计实际上代替了对应用的抽象，是我们可以使用该 Deployment 来描述应用。

## Reference

1. [Kubernetes --- Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
2. [极客时间 --- 深入剖析 Kubernetes](https://time.geekbang.org/column/article/40906)
3. [详解 Kubernetes Deployment 的实现原理](https://draveness.me/kubernetes-deployment/)