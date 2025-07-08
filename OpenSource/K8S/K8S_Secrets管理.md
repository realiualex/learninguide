# K8S Secrets 管理的方式

## K8S 原生的 Secrets 

如果使用 K8S 原生的Secrets，那么secrets将通过base64编码的方式存在 secrets中，这样不是很安全，尤其是无法将 secrets 的配置信息推送到 git 仓库里

## 使用云厂商的 sercrets manager或 key vault

在使用云厂商的secrets manager，或者 hashcorp的 key vault等产品的时候，可以通过[External Secrets Operator (external-secrets.io)](https://external-secrets.io/main/) 这个operator来储存secrets，这样yaml文件中，只需要指定云厂商的secerts 名字就可以了，云平台更新了 secrets 值，这个operator也会更新到 k8s 集群(可能有几秒的延迟)

## 使用 bitnami 的 sealed-secrets

如果没有用外部的 secrets manager，只有 k8s 的情况下，也可以通过 bitnami 的sealed secrets 将secrets 进行加密，这样只有在当前集群内才能解密这个secrets，将加密的secrets 放在其他地方也无法解密。

具体操作：需要在 k8s 集群中，安装sealed-secrets这个operator，同时本地安装kubeseal。在本地使用 kubectl create secrets的时候，通过管道和 kubeseal 对内容进行加密，生成新的 yaml文件，将这个yaml文件可以存储到 git仓库里，也无需担心其他人看到数据。然后再通过 argoCD 或 Flux 来做CD，部署到 K8S 里之后，就是解密后 base64编码的secrets 了

[GitHub - bitnami-labs/sealed-secrets: A Kubernetes controller and tool for one-way encrypted Secrets](https://github.com/bitnami-labs/sealed-secrets)

