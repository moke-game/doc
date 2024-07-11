# k8s 集群管理和监控

k8s是用于自动部署、扩缩和管理容器化应用程序的开源系统。

## 为什么要使用k8s

* 对于无法预估规模的应用/游戏，k8s可以自动扩缩,应对不同的压力场景
* 提供了各种部署策略：滚动更新，蓝绿部署，金丝雀部署等等，以实现服务的不停服更新
* 拥有一系列特性：自动化上线和回滚，自动扩缩，服务发现和负载均衡，存储编排，自动部署等等
* 有大量的社区支持和插件，可以满足各种需求
* 可以在各种云厂商上部署，比如aws，gcp，阿里云等等
* 稳定性高，已经在很多大公司得到验证

## 需要提前安装

* [AWS CLI install](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-configure.html)
* [kubectl install](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/create-kubeconfig.html)
* [Helm install](https://helm.sh/docs/intro/install/)
* [k9s](https://k9scli.io)

## k8s的集群创建流程和管理

这里介绍如何在aws上部署k8s集群

1. 创建[aws EKS](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/create-cluster.html)
2. [Amazon EBS CSI install](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/ebs-csi.html)
3. 创建[NodeGroup](https://docs.aws.amazon.com/zh_cn/eks/latest),并配置IAM权限策略
4. 配置权限:
   ```shell
   aws configure // 配置aws cli 需要相关权限文件
   aws eks update-kubeconfig --name <your cluster name> --region ap-southeast-1
   ```
5. 推荐安装[k9s](https://k9scli.io/topics/install/)来管理集群服务
   ```shell
    winget install k9s // windows 安装k9s
    k9s // 查看集群状态
    ```

## 服务安装

* install with [helm](https://helm.sh/)
   ```shell
   # create a helm demo
    helm create demo
    # install demo in k8s
    helm install demo ./demo
    # delete demo
    helm delete demo
  # list all helm package
    helm list
   ```

* install with [kubectl](https://kubernetes.io/docs/reference/kubectl/)
   ```shell
   # you should create a k8s yaml file, and apply it
   kubectl apply -f ./demo.yaml
  # delete demo
    kubectl delete -f ./demo.yaml
   ```

## 集群访问

* 本地开发可以使用forward方式访问集群中的服务
  ```shell
  # 运行命令后你就可以在本地端口:8080访问到k8s中的服务了
  kubectl port-forward svc/demo 8080:80
  ```
* [ingress方式](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/),已经停止更新
  ```shell
  # install ingress-nginx
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update 
  # 你需要创建自己的 .\ingress-nginx\values.yaml 文件,或者使用默认配置也行
  helm install nginx -f .\ingress-nginx\values.yaml ingress-nginx/ingress-nginx 
  ```
* [gatewayAPI](https://kubernetes.io/zh-cn/docs/concepts/services-networking/gateway/),新的替代方案

## 集群监控
 推荐使用[grafanaLabs云服务](https://grafana.com/solutions/kubernetes/)监控k8s集群:
* 自带了日志(loki)，性能指标(prometheus)，链路追踪(tempo)等服务，可以满足大部分需求。
* 有大量的插件和[dashboard模板](https://grafana.com/grafana/dashboards/)，可以直接使用,非常方便
* all in one,不需要自己搭建各种服务，直接部署收集器到k8s集群即可
* 可以自定义报警规则，方便监控
* 免费版有限制，但对于小团队来说已经足够了
 