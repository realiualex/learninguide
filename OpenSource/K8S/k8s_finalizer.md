当我们想要对 k8s 的资源设置删除保护的时候，可以用`.metadata.finalizers` 的属性，这样资源不能被删除，但可以被更新。比如我们有一个deployment，担心误删除，但却会对这个资源进行更新，就可以用这样的功能.

示例:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: awscli-app
  finalizers:
    - protection.test.io/finalizer
  name: awscli-app
  namespace: devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: awscli-app
  template:
    metadata:
      labels:
        app: awscli-app
    spec:
      containers:
      - name: awscli-app
        image: amazon/aws-cli:2.31.25
        command: ["sleep", "infinity"]

```

当我们通过上述方式创建了 deployment，之后执行 `kubectl delete deployment` 的时候，就会被卡住。此时查看这个deployment 的信息，会发现 .metadata.deletionTimestamp 有一个被标记删除的时间 ( 这个时间是首次执行 delete deployment 的时候的时间，如果再次执行delete动作，并不会更新这个时间).
但是当我们移除了这个 finalizers的标记，这个deployment就会立即被删除( 无需再次执行 delete deployment)
```shell
kubectl patch deployment awscli-app -p '{"metadata":{"finalizers":[]}}' --type=merge
```

需要注意的是：
如果我们的 deployment 已经设置了 finalizer，此时假设在部署的yaml文件里，移除 finalizer，再次进行部署。那么实际会删除原有的deployment，而不会创建新的deployment。这个一定要小心

