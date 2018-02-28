## 创建集群

多数 Kubernetes 安装方法都没有提供[网络策略](https://kubernetes.io/docs/concepts/services-networking/network-policies/)的支持，因此需要手工安装 Weave Net 或者 Calico 这样的网络策略支持工具。

**[Google Kubernetes Engine (GKE)][gke]** 让用户可以在不进行手工安装的情况下轻松的获得网络策略支持，这是因为 GKE 已经选择 Calico 作为网络支持组件了（这个特性在 Kubernetes 1.8 中处于 Beta 阶段）。

下面的命令会在 GKE 中创建名为`np`，并支持网络策略功能的集群：

~~~
gcloud beta container clusters create np \
    --enable-network-policy \
    --zone us-central1-b
~~~

命令执行后，会生成一个三节点的 Kubernetes 集群，并启用网络策略能力。

完成教程之后，可以用下面的命令删除集群：

~~~
gcloud container clusters delete -q --zone us-central1-b np
~~~

[gke]: https://cloud.google.com/kubernetes-engine/
