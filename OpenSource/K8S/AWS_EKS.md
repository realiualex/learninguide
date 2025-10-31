## irsa 和 pod identity 

### irsa
EKS 里的pod，获得AWS平台的能力，以前都是通过 IRSA 的方式，即让pod 绑定一个service account，这个service account 是 EKS OIDC 提供的 jwt token，IAM Role信任了这个 OIDC，然后通过 AssumeRoleWithWebIdentity 的方式，拿到 AK/SK/Token。在一个大型公司里，一般有 IAM admin以及 业务k8s admin。按照这类方式，我们一般会在 iam role condition里，对jwt token 的 aud 做一些过滤，确保只有特定的 k8s service account 才能assume 到这个role。此时每次只要 k8s 环境有新的服务部署，有新的service account 产生，就要找 iam admin。非常不方便

### pod identity
* 从使用者角度看：
pod identity 简化了irsa的设置。在pod identity的场景下 (要先安装 pod identity daemonset)，iam role 信任的不再是 oidc，而是 `pods.eks.amazonaws.com` 这个服务。然后 eks admin 可以通过 `aws eks create-pod-identity-association` 这个api，将一个 service account 和 iam role绑定，之后创建 deployment 的时候，指定这个 service account 就可以了。

* 从原理层看:
其实 pod identity 的本质，还是 service account 的jwt token，只是以前是用户的服务(aws sdk) 直接跟 sts 服务交互，现在变成了将这个 jwt token 发给了 eks 服务，由eks 服务代用户请求 sts assume role 而已。

具体说来：
在正常 使用pod identity 的时候， 在 eks-pod-identity-agent 容器内执行的是 aws eks-auth 的接口而不是 sts接口换 aksk。但是 eks-auth 的服务，在收到了这个API请求后，还是通过 sts assume-role-with-web-identity 的方式，找 sts 服务换的 aksk，这个是帮用户做了，用户不可见。
eks-auth 执行的 cloudtrail 记录 event name是 `AssumeRoleForPodIdentity`，而不是 `AssumeRoleWithWebIdentity`

验证:
pod identity 的 jwt token，其实也可以直接拿到外面用传统的 assume role with web identity 的方式换出来 aksk，即由两种方式(当然 pod identity在 daemonset里用的是方法二，方法二隐性的完成了方法一)：

方法一： 
```shell
aws sts assume-role-with-web-identity --role-arn ROLE_NAME --web-identity-token POD_IDENTITY_JWT_TOKEN
```

方法二:   
```shell
aws eks-auth assume-role-for-pod-identity --cluster-name EKS_CLUSTER_NAME --token POD_IDENTITY_JWT_TOKEN
```