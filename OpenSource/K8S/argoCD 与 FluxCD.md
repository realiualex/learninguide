
## FluxCD
FluxCD 需要安装在业务集群内部，可以与一个仓库的某一个路径绑定，这样在这个仓库的路径及其子路径里，只要有yaml文件产生，就可以自动 apply 这些yaml，比较简单。

## argoCD 
ArgoCD 可以安装到非业务集群里，也可以安装到业务集群里。所以argoCD在使用的时候要添加的集群，并且创建argoCD application，而不会自动扫描某一个路径自动apply所有yaml.
但是ArgoCD 提供了 ApplicationSet 功能，该功能也可以根据目录结构，自动生成Application，也能实现和 FluxCD 类似的效果.

### AWS EKS ArgoCD capability
1.  如果创建private argoCD url，那么要创建 VPC Private Link Endpoint （eks-capabitlity），安全组要开放 argoCD客户端的网络（如果是VPN打通，目前有bug，要改本机 host文件，解析到这个VPCE IP）
2. 要改EKS集群的安全组，允许 private link endpoint访问 eks api的443端口
3. 创建好 arogocd capability（如果 eks 要拉ghe 的代码，role 要有 `codeconnections:UseConnection` 权限，拉ecr 的话，要有ecr权限）
4. 注册集群到 argocd里 (也可以用 argocd cli 注册，但argocd cli既要练 eks，又要一个注册token)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: qa-eks-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  name: qa_eks_cluster
  server: arn:aws:eks:ap-northeast-1:123456789012:cluster/qa_eks_cluster
  project: default
```

5. 创建application。注意 argoCD 和 fluxcd不同，fluxcd 只需要指定仓库的路径，可以自动apply yaml。但是 argocd 要对每一个应用创建一个application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-first-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://codeconnections.ap-northeast-1.amazonaws.com/git-http/1234567890/ap-northeast-1/1a2b3c4d5e-e7f7-41c2-bf66-bf370388454f/ORG_NAME/REPO_NAME.git
    targetRevision: master
    path: eks-clusters/applications/my-first-application
  destination: 
    namespace: argocd
    server: arn:aws:eks:ap-northeast-1:123456789012:cluster/my-eks-cluster
  syncPolicy:
    automated:
      prune: true
    syncOptions:
      - CreateNamespace=true

```

