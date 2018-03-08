# 拦截命名空间内所有白名单之外的 Egress 流量

## 用例：

这是一个基础策略，缺省屏蔽命名空间内所有外发（egress）流量，部署之后，还可以部署白名单策略来允许特定的外发流量。

可以考虑将这一规则应用到所有业务负载命名空间。

## 最佳实践

这个策略可以作为缺省策略，拒绝所有流量。这样就可以清楚的识别出组件之间的依赖，利用网络策略的定义就能清楚的勾画出组件间的依赖图。

## default-deny-all-egress.yaml

~~~yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny-all-egress
  namespace: default
spec:
  policyTypes:
  - Egress
  podSelector: {}
  egress: []
~~~

上面文件中的一点细节：

- `namespace: default`：部署到`default`命名空间。
- `podSelector:`：为空，就是匹配命名空间的所有 Pod。
- `egress`：规则列表为空，这会导致来自`default`命名空间内的 Pod 发出的包括 DNS 查询在内的所有外发流量被禁止。

保存文件，应用到集群上：

~~~sh
$ kubectl apply -f default-deny-all-egress.yaml
networkpolicy "default-deny-all-egress" created
~~~

## 清理

~~~sh
kubectl delete networkpolicy default-deny-all-egress
~~~
