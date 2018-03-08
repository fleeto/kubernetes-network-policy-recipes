# 拦截外部的 egress 流量

## 用例

- 禁止某些应用向外部网络发起连接。

> **注意**：如果你正在使用的是 GKE，至少需要`1.8.4-gke.0`的 Master 和 Node 才能使用 egress 策略。

## 例子

下面的策略保存为：`foo-deny-external-egress.yaml`

~~~yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: foo-deny-external-egress
spec:
  podSelector:
    matchLabels:
      app: foo
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  - to:
    - namespaceSelector: {}
~~~

这条策略的一些说明：

- 应用到`app=foo`的 Pod，以及外发通信。
- 和前面一篇情况类似，允许 DNS 查询。
- `to:`的值是一个空的`namespaceSelector`，代表着所有命名空间的所有 Pod。所以到集群内的 Pod 的通信请求是可以通过的。
  - 因为没有列出，所有到集群外的 IP 的通信请求都会被禁止。

保存 YAML 文件并应用到集群：

~~~sh
kubectl apply -f foo-deny-egress.yaml
networkpolicy "foo-deny-egress" created
~~~

## 测试

创建`app=web`的 Web 应用：

~~~sh
kubectl run web --image=nginx --port 80 --expose \
  --labels app=web
~~~

运行一个`app=foo`的 Pod，我们的策略会对这个 Pod 生效：

~~~sh
$ kubectl run --rm --restart=Never --image=alpine -i -t -l app=foo test -- ash

/ # wget -O- --timeout 1 http://web:80
Connecting to web (10.59.245.232:80)
<!DOCTYPE html>
<html>
...
(connection is allowed)
~~~

说明`app=foo`标签的 Pod 是可以连接到`web`服务的

下面测试一下集群外的请求：

~~~sh
/ # wget -O- --timeout 1 http://www.example.com
Connecting to www.example.com (93.184.216.34:80)
wget: download timed out
(connection is blocked)

/ # exit
~~~

Pod 成功解析了`www.example.com`的 IP 地址，但是连接无法建立，证明我们的策略的确阻止了集群外的通信。

## 清理

~~~sh
kubectl delete deployment,service web
kubectl delete networkpolicy foo-deny-external-egress
~~~
