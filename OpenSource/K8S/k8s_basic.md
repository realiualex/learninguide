# K8S 基础

## kubectl cheetsheet

* kubectl command   
  https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#autoscale


* 直接创建pod(在先前版本的k8s创建的是deployment)  
  
   ```
   kubectl run nginx --image=nginx
   ```
   
* 创建pod的时候指定资源限制，如果只指定limits，不指定requests，那么requests就和limits一样。但是如果指定了requests，没有指定limits，则没有limits限制。  
  
  ```
  kubectl run nginx-pod --image=nginx --limits cpu=200m,memory=512Mi --requests 'cpu=100m,memory=256Mi'
  ```
  
* 直接创建 deployment  
  
  ```
  kubectl create deployment nginx --image=nginx
  ```
  
* 显示pod的label

  ```
  kubectl get pod --show-labels
  ```

* 给 pod 打label

  ```
  kubectl label pod nginx-test-6557497784-2wwsq labeltest=labelvalue
  ```

* 暴漏pod的端口为load balancer  
  
  ```
  kubectl expose pod nginx --port 80 --target-port 80 --type LoadBalancer
  ```
  
* 暴漏deployment为load balancer  
  
  ```
  kubectl expose deployment nginx-deployment --port 80 --type LoadBalancer
  ```
  
* 使用HPA自动扩容pod(即使deployment没有指定资源limit也能创建hpa，但是不工作的)  
  
  ```
  kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=3 --max=10
  kubectl get hpa
  ```
  
* 手动扩容
  
  ```
  kubectl scale deployment nginx-deployment --relicas=2
  ```
  
* 在使用 containerd 的k8s节点上，如果想要列出容器：
  
  ```
  # -n 指定 namespace，k8s的容器namespace是 k8s.io
  ctr -n k8s.io containers list
  # 列出所有 namespace
  ctr ns ls
  # 列出容器相关任务
  ctr -n k8s.io tasks list
  ```
  
* 在 containerd 的k8s 节点上，如果想要看到某一个容器的日志，比较麻烦，方法为:

  ```
  # my-namespace_my-pod-uid 和 container-name 可以通过 bash的tab 补全查看
  tail -f /var/log/pods/my-namespace_my-pod-uid/container-name/0.log
  ```

* 列出k8s所有的API，包括 CRD API
```
kubectl api-resources
```

* 检查 k8s 版本，包括客户端版本和服务端版本
```
kubectl version
```

* 检查当前用户
```shell
kubectl auth whoami
```
``
* 访问api，比如获得jwts信息
```shell
kubectl get --raw /openid/v1/jwks
```

##  k8s prune
以前在k8s 1.5的时候， kubectl 就支持 --prune 参数，但当时是靠label来实现的，会遍历包含这个label的所有资源，看是否跟 yaml文件里的match，如果发现有这个label的资源，但yaml里没有，就会删除。
在 [k8s 1.27](https://kubernetes.io/blog/2023/05/09/introducing-kubectl-applyset-pruning/) (2023年5月份)的时候，支持了 KUBECTL_APPLYSET的功能，在创建资源的时候，需要启用 APPLYSET以及加上 --prune，此时资源会被自动打上特定label。之后再次apply的时候会进行对比。
```shell
KUBECTL_APPLYSET=true kubectl apply -f <directory/> --prune --applyset=<name>
```

除了k8s自带的这个功能之外，argoCD 也能实现类似的功能，也是靠自动打label实现的，这样argoCD就能知道哪些资源是argoCD创建的。argoCD如果发现 k8s里的资源状态，跟仓库里不一致，会标记为 不一致，也可以启用自动sync为仓库里的状态(需要手动启用)。

## nodeSelector 和 nodeAffinity
nodeAffinity 是升级版本的 nodeSelector，都是选择任务调度到哪个node上的。nodeSelector比较简单，也容易写，但不支持复杂的逻辑，比如OR、Exists、DoesNotExist、In、NotIn运算符等(只支持AND)。nodeAffinity比较复杂，支持逻辑运算符，但逻辑比较复杂，没有nodeSelector那么简单。  
简单来讲，nodeSelector是“必须满足”，而 nodeAffinity是“更聪明的选择节点”（比如优先选择A节点，如果A没有，则选择B节点，并且B节点不是测试环境）。也许未来有一天，nodeSelector会被nodeAffinity完全替代

## taint 和 tolerations 
虽然 nodeSelector 或 nodeAffinity 已经能够实现节点的选择，可是有时候，我们想要实现一些特殊的功能。比如我有不同的节点组，节点组A是给业务A用的，节点组B是给业务B用的，节点C是给运维工具用的。A和B由于都是业务，所以业务yaml都是自己写的。但安装运维工具的时候，很多运维工具yaml是互联网上其他人提供的，我们没法指定 nodeSelector 或 nodeAffinity。此时我们可以给节点组 A 和节点组 B 打上污点taint，这样运维工具就不能安装到节点组 A 和 B 上了。然后部署业务 A服务的时候，我们可以用 tolerations 来允许污点 A，同样部署B服务的时候，用tolerations 允许污点B，从而实现节点任务的调度。  
但并不是说，有了 taint 和 tolerations 就不需要 nodeSelector/nodeAffinity 了，如果在节点组A里，有更复杂的逻辑，想要实现某一个服务，在节点组A的指定机器上，就是需要 nodeSelector / nodeAffinity 的.

## overhead 与 request/limit
在创建pod的时候，可以指定 request 和 limit 的cpu 与memory，比如cpu 为100m，指的是1个逻辑cpu的0.1 （1000为1个逻辑cpu），其中request指的是要求这个pod被调度的机器上，至少有 request要求的资源，limit指的是这个pod里的进程最多用的资源。  
注意request 和limit，指的是pod里的进程用的资源，但实际pod在启动的时候，pod本身也会占用一定的资源，比如pod的网络插件，log插件等等。而为了管理pod所占用资源的定义，就是 overhead。所以当pod进行调度的时候，要求node上的资源至少满足 request + overhead 之和。  
需要注意的是，overhead是在 RuntimeClass 上定义的，而并非在pod上定义。一般集群管理员统一设置 overhead，在创建pod的时候，通过 spec.runtimeClassName选择这个runtime class，不能直接把 overhead写到pod的定义里，如果在pod的yaml里直接写overhead，kubnernetes的准入控制器会拒绝这个pod的创建。  
没有办法设置默认的 Runtime Class，也就意味着，k8s 集群里，可以有多个Runtime class，如果pod没有指定，则没有启用overhead检查，pod在调度的时候，只按 request 的 cpu/memory 进行调度，而并不会把 request 和 overhead 加一起作为判断标准.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
overhead:
  podFixed:
    memory: "120Mi"
    cpu: "250m"
```

pod 规范里只需要这样写就能引用这个配置
```yaml
spec:
  runtimeClassName: kata-fc
```

## 常用测试的yaml

* [创建deployment](nginx-dep.yaml)
* [创建svc](nginx-dep-svc.yaml)
* [创建hpa自动扩容](nginx-dep-hpa.yaml)
  

## k8s 证书管理

* 通过cert-manager来管理证书  
* cert-manager 是通过CRD(custom resource defination) 来管理证书， 会起几个cert-manager的pod，如果遇到问题，可以排查这几个pod状态。
  https://cert-manager.io/docs/installation/kubernetes/
  <br />

* 通过let's Encrypt自动申请证书。申请之后，可以用 kubectl get certificate 来查看申请到的证书，状态为Ready时表示可用
  https://docs.microsoft.com/en-us/azure/aks/ingress-tls
  <br />

* 故障排查
  
 ``` 
 kubectl get certificate
 kubectl get certificaterequest
 kubectl get clusterissuers
 ```


## 切换kubeconfig

* 默认kubeconfig保存在 ~/.kube/config 文件中，里面可以存多个集群的信息。
  
  ```
  kubectl config current-context
  kubectl config use-context CONTEXT_NAME
  kubectl config get-contexts
  kubectl config get-clusters
  ```

## 管理coreDNS

在先前k8s版本中，使用的是kube-dns，在k8s 1.12之后被CoreDNS替代。coreDNS是通过 [Corefile](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/) 管理的，可以通过利用[hosts plugin](https://coredns.io/plugins/hosts/)编辑coredns的configmap来添加自定义dns条目   
```
kubectl edit configmap coredns -n kube-system
```

在corefile中添加下面的几项   
```
    example.org {
      hosts {
        11.22.33.44 www.example.org
        fallthrough
      }
    }
```

然后可以起一个测试dns的pod来测一下解析
```
 kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```

不过在云平台托管的k8s中，如Azure的AKS中，无权限编辑 Corefile，以Azure为例，Azure提供的aks的coredns的deployment中，添加了对挂载的volume的支持，通过查看coredns的deployment可以看到，默认挂载了一个叫做 coredns-custom的ConfigMap，因此我们可以创建一个名为 coredns-custom的configmap，在这个configmap中来添加自定义DNS解析的条目.

首先先看一下默认的配置:  
```
kubectl describe deployment coredns -n kube-system

  Volumes:
   config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      coredns
    Optional:  false
   custom-config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      coredns-custom
    Optional:  true
   tmp:
```

添加自定义解析，将 www.example.org 解析到 11.22.33.44，需要注意的是：configmap的名字必须为 coredns-custom，这样CoreDNS才能识别，其他名字无法识别   
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom # this is the name of the configmap you can overwrite with your changes
  namespace: kube-system
data:
    test.override: | # you may select any name here, but it must end with the .override file extension
          hosts example.org { 
              11.22.33.44 www.example.org
          }
```

之后需要重启coreDNS才能生效  
```
kubectl rollout restart -n kube-system deployment/coredns
```

## 安全相关组件

### secrets 与 configmap

虽然 k8s secrets 与configmap 都是明文的(secrets是 base64 encode)，但实际上，如果有比较敏感的数据，还是建议放在secrets里，主要有几点：1. 更容易做 RBAC权限控制，2. configmap会记录在日志中，3. secrets 存在ETCD里也容易做隔离

### service account

在创建服务的时候，推荐每个服务都绑定一个自己的 service account，尤其是我们看外部其他人的项目的时候，会经常发现创建一个空的 service account，然后绑定到容器上。主要是为了：1. 万一以后需要某个服务访问k8s api，这样容易控制 2. 如果不指定service account，那就意味着使用默认的service account，容易误操作给默认的service account赋予不必要的权限，造成漏洞

## K8S 的坑

* 使用 kubectl apply的时候，如果资源名发生变更，其不会删除旧的资源，会创建一个新的，需要手动删除。使用helm就没这个问题，能更好的管理应用的生命周期

## kubeadm 故障排查
在使用 kubeadm 搭建k8s，使用 containerd 的时候，如果遇到安装了 网络插件，如flannel之后，node依然NotReady.
describe node报错"container runtime network not ready:NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"。
在node节点上，看containerd 的日志  journalctl -u containerd | grep cni，报错 
"Dec 29 05:46:20 ip-10-79-41-224.ap-northeast-1.compute.internal containerd[2862]: time="2024-12-29T05:46:20.428267622Z" level=info msg="No cni config template is specified, wait for other system components to drop the config.""

此时应该是 containerd 没有启用CNI插件导致的。需要在 /etc/containerd/config.toml 里加一行

```toml
[plugins."io.containerd.grpc.v1.cri".cni]
  bin_dir = "/opt/cni/bin"
  conf_dir = "/etc/cni/net.d"
  max_conf_num = 1
```