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

## k8s 的 oidc 认证
对于云厂商来说，很多是将 k8s service account 创建的 jwt token，通过 assume role with web identity 的方式，跟云平台交换出来 ak sk，使pod内能够获得云平台cred的能力。
这里需要说明的是：
* 只要是 k8s service account，k8s 就会将其生成 jwt token，在 k8s 1.24之前，这个 jwt token是长期存在 k8s secrets里，但是1.24以后，就需要用户挂载为 volume，放在pod指定路径下，k8s自己刷新这个jwt token，变成短期token。
* 在k8s没有开启OIDC能力的时候，k8s 的service颁发的jwt token，只能用于k8s集群内的认证，因为只有 k8s api server能验证这个 jwt token的有效性
* 如果想要把jwt token放到k8s外部验证，就需要k8s开启 OIDC的能力了
* 获得 k8s jwt token内容的方法:

```shell
kubectl get --raw /openid/v1/jwks
```
### oidc token
一般我们经常听到 id token 和 access token，这两个token 都是jwt 格式的，但用途不同。id token是做 authentication认证的，而 access token 是做 authorization 授权的。具体说来
* id token: 这个token里存的是用户的信息，比如 username, email, profile 等等信息，这样你的网站，可以拿到第三方oidc的身份提供商的用户信息，在网站上做展示。甚至verify了这个id token信息之后，就可以直接登录成功。
* access token：这个token是用来请求颁发这个token的api server的接口的。比如网站A，用了 google oidc 登录，那么网站A应该只用 google的 id token，获得用户的身份信息，假设网站A的api 接口，也是jwt token格式，那么网站A的接口，应该用网站A自己颁发的 access token，而并非google 的access token
* refresh token: 一般来说，access token由于请求的时候，会在互联网上传递，所以有被黑客盗取的风险，那么我们就把 access token的有效期设置短一点。客户端在向服务器申请 access token 的时候，一些服务器会把refresh token也返回过来，客户端可以将refresh token保留到本地。一旦access token 过期，不需要用户认证，拿这个fresh token换一个新的access token就可以了。需要注意的是：access token 是在网络传输的时候被盗取，而refresh token依然存在本地被盗取的风险（即使access token过期，只要refresh token本地还存在，黑客就能交换出access token）


## cpu pinning
如果 K8S 的节点配置比较高，有很多cpu核心，那么在这个节点启动的容器，有可能会不停的在不同的CPU上进行切换，甚至是不同的NUMA节点之间切换，导致上下文切换过多和缓存未命中，从而增加延迟，尤其是对cpu调度比较敏感的任务，cpu pinning可以为其绑定cpu核心，减少上下文切换次数。  
默认情况下，kubelet 可能并未启用 cpu pinning，这个需要k8s worker node里的 kubelet 进程在启动的时候，加上 --cpu-manager-policy=static 参数才能启动。如果是AWS EKS，需要自定义 launch template，然后在 EC2 userdata里，通过脚本改 kubelet 的这个参数(以前是 /etc/eks/bootstrap.sh，现在改成了 nodeadm 程序).  
在 kubelet 开启了 cpu-manager-policy=true 的 k8s 节点上（ v1.26+， 2022年12月发布），必须满足下面条件，启动的容器才会跟 cpu 核心绑死。
* Pod QoS 必须是 Guaranteed(即 cpu request 和 limit 一样)，不能是 Burstable 或 BestEffort
	* 如果Pod里容器不设置 request 和limit，QoS 是 BestEffort
	* 如果 pod 的每个容器的request 和limit 都是一样的，QoS 就是 Guaranteed
	* 如果Pod里，至少有一个容器设置了request(没有要求设置 limit)，QoS就是 Burstable (如果Pod里有2个容器，一个设置了一样的limit 和 request，另一个什么也没设置，此时也是 Burstable，因为 Guaranteed 要求每个容器的 limit 和 request 都一样)
* 除了 CPU 的request 和 limit 要一样之外，必须是整数值 (如 "cpu": "2")，不能是 100m 这类 millicore ("cpu": "100m")

如果一个k8s节点开启了 cpu pinning, 假设这个节点有4个vcpu，此时有一个pod满足cpu pinning的要求，假设申请了 2个vcpu，此时跟 cpu 0 和 cpu 3 绑定了。cpu 0 和cpu 3 就会从共享池中移除，后续调度到这个节点的pod，只能用 cpu 1 和 cpu 2（无论后续pod是否满足 cpu pinning），无法跑在 cpu 0 和 cpu 3 上


## Pod Priority 和 Preemption(抢占)
默认情况下，k8s 有两个 priority class，分别是 system-node-critical 和 system-cluster-critical，其中 system-node-critical 的优先级更高一些（数字越大，优先级越高）。在创建 Pod 的时候，在pod的 .spec 下可以通过 priorityClassName 来指定使用哪个 priority class。如果没有指定 priority class，那么pod 的优先级是0，即为最低的。此时只要指定了priority class的pod，如果资源不够，就会杀死没有指定priority class的，然后自己调度过来。
由于系统默认的 system-node-critical  高于system-cluster-critical，所以如果一个机器上，资源只够起1个pod，假设第一个pod的 priority class name是 system-cluster-critical，那么起第二个pod的时候，假设为 system-node-critical ，就会杀死第一个pod，将第二个pod启动。

preemption 是 priority class里的一个功能，默认情况下 preemptionPolicy 是 PreemptLowerPriority，也就是如上述所说，会驱逐低优先级的pod。但如果你不想让它自动驱逐，想要让它等待系统有资源的时候，优先抢占这个资源，那么可以将 preemptionPolicy 设置为 Never
