# 深入了解 K8S

## Pod的生命周期

1. 当用户发起删除Pod的命令时，默认会有30秒的宽限期

2. 如果超过宽限期，API Server会将Pod状态标记为 dead

3. 客户端命令行显示的Pod状态为 terminating

4. 在上面第三步的同时，kubelet发现pod被标记为 terminating，然后会发送 SIGTERM信号停止Pod

   1. 如果Pod定义了 preStop Hook，在停止Pod前会被调用。如果宽限期过了，preStop hook依然存在，第二步会增加2秒的宽限期
   2. 之后向Pod发送 TERM信号

5. 在上面第三步的同时，Pod会从service的Endpoint列表中移除，不再是Replication Controller的一部分，关闭慢的Pod依然会继续处理LoadBalancer的流量

6. 过了宽限期，会向Pod中依然运行的进程，发送 SIGKILL 信号而杀死进程

7. kubelet会在API Server中完成Pod的删除，通过将 grace period 设置为0 (立即删除)，Pod在API Server中消失，并且在客户端也不可见

   修改默认的宽限期，可以在 kubectl delete --grace-peroid=\<second> 中指定，如果要设置为0立即删除，还要加上 --force参数。yaml文件中是在 \{\{.spec.spec.terminationGracePeriodSeconds \}\} 指定

### Pod 的 lifecycle hook

容器支持两个[lifecycle hook](https://kubernetes.io/zh-cn/docs/concepts/containers/container-lifecycle-hooks/)，一个是 PreStop，指的是在容器stop之前的hook，另外一个是 PostStart，指的是容器被创建后立即执行(但不能保证执行PostStart的命令会在ENTRYPOINT之前执行)。

有关具体用法，可以参考这个示例

```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

## 有关Pause/Infra Container

一个 Pod 里面的所有容器，它们看到的网络视图是完全一样的。即：它们看到的网络设备、IP 地址、Mac 地址等等，跟网络相关的信息，其实全是一份，这一份都来自于 Pod 第一次创建的这个 Infra container。这就是 Pod 解决网络共享的一个解法.

在 kubetlet的启动参数中，有一个 --pod-infra-container-image 的参数，后面跟了一个 pause container 的镜像，这个pause container是在Pod中最先启动的，会申请network namespace的资源，然后该pod里的其他容器，会通过join namespace的方式，加入到这个namespace中。具体加入的方法是:

```
docker run -d --name nginx --net=container:{PAUSE_CONTINER_NAME OR PAUSE_CONTAINER_ID} --ipc=container:{PAUSE_CONTINER_NAME OR PAUSE_CONTAINER_ID} --pid=container:{PAUSE_CONTINER_NAME OR PAUSE_CONTAINER_ID} nginx
```

在K8S 的worker node上，使用 docker inspect检查应用容器，在NetworkMode中，能看到对应的pause container的容器信息，不过pause container本身，NetworkMode是空的



## PDB: Pod Disruption Budget (干扰预算)

在对节点维护之前，一般往往会先用 kubectl dordon 禁止pod调度到该节点，然后 kubectl drain 来排空节点，在排空的时候，如果某一个 deployment 的所有 pod 都在这个节点上，或者大多数 pod 都在这个节点上，在这个过程中，可能会出现问题。一般最佳实践使使用 [pod disruption budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)，pdb 将限制在同一个时间段内自愿中断的应用程序中断的pod的数量，我们可以设置 .spec.minAvailable 或 .spec.maxUnavailable 来指定只要要有多少个available pod 或者 最多有多少个 unavailable pod. 示例:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper
```



## label 选择

在创建某个资源的时候，可以通过 selector 来选择资源，可以通过 match label来强匹配，也可以用 matchExpressions 来通过某个表达式来过滤

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```



## 重新调度

如果 K8s 的节点负载不均衡，比如新增加了节点，或者之前在维护的时候，对节点上的pod做了驱逐，想要重新balance下，可以用 [descheduler](https://github.com/kubernetes-sigs/descheduler) 这样的插件


## cloud secrets 管理
由于 k8s secrets 是明文存储的，并不是特别安全，所以一般会将secrets 存在外部，比如 AWS Secrets Manager, Azure Key Vault, HashiCorp Vault里等，在使用的时候，需要创建一个 service account，将 cloud secret 跟k8s 里的service account进行绑定，之后再创建SecretProviderClass，以及创建容器pod或 deployment，在deployment里，指定 serviceAccount，以及Volume，volume里需要指定csi driver以及 volumeAttributes，这些都是k8s sig 里的标准用法，云厂商只需要在 SecretProviderClass里实现 secret与 k8s volume csi driver之间的对接就可以了。
一般来说，都是通过volume挂载到 k8s 的某一个文件路径，不过也可以在 SecretProviderClass 的 .spec.secretObjects 里指定映射为 k8s 里标准的secret，然后在 Deployment 的 .spec.containers[0].env.valueFrom.secretKeyRef 里，引用这个对象
一个完整华为云CCE的示例（即包含了映射为env，也包含了映射为volume，实际使用可以只用其中一种方法）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alex-secrets
  namespace: alex-test
  annotations:
    cce.io/dew-resource: "[\"alex-secrets\"]"  #secrets that allow pod to use

---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: alex-spc-test
  namespace: alex-test
spec:
  provider: cce     # The value is fixed at cce.
  parameters:
    objects: |
          - objectName: "alex-secrets"
            objectVersion: "latest"
            objectType: "csms"
  secretObjects:
    - secretName: alex-secrets-env
      type: Opaque
      data:
        - objectName: alex-secrets
          key: alex-secrets

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alex-nginx
  namespace: alex-test
spec:
  selector:
    matchLabels:
      app: alex-nginx
  template:
    metadata:
      labels:
        app: alex-nginx
    spec:
      containers:
      - name: alex-nginx
        image: nginx:latest
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
          requests:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
        volumeMounts:
          - name: alex-secrets-volume
            mountPath: "/mnt/secrets-store"
            readOnly: true
        env:
          - name: ALEX_SECRET_ENV
            valueFrom:
              secretKeyRef:
                name: alex-secrets-env
                key: alex-secrets
      serviceAccountName: alex-secrets
      volumes:
        - name: alex-secrets-volume
          csi: 
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "alex-spc-test"


```