# 拦截到一个应用的所有流量

这一策略使用 Pod 选择器来进行 Pod 选择，会拦截所有的目标为某应用的 Pod 的流量。

## 用例

- 非常普遍：如果以白名单的方式来使用网络策略，首先要使用这一策略来进行拦截。
- 想要运行一个不被任何其他 Pod 访问的 Pod。
- 需要暂时性的进行来自其他 Pod 的流量的隔离。

![Diagram for DENY all traffic to an application policy](img/1.gif)

## 示例

使用`app=web`标签运行一个 Nginx Pod，并暴露其 80 端口：

~~~sh
kubectl run web --image=nginx --labels app=web --expose --port 80
~~~

创建一个临时 Pod，在这一 Pod 中访问`web`服务：

~~~sh
$ kubectl run --rm -i -t --image=alpine test-$RANDOM -- sh
/ # wget -qO- http://web
<!DOCTYPE html>
<html>
<head>
...
~~~

运行成功之后，我们把下面的内容保存为`web-deny-all.yaml`，并将其应用到集群上。

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-deny-all
spec:
  podSelector:
    matchLabels:
      app: web
  ingress: []
```

```sh
$ kubectl apply -f web-deny-all.yaml
networkpolicy "access-nginx" created
```

## 测试

再次运行一个测试容器，在这一容器中访问上面建立的服务：

~~~sh
$ kubectl run --rm -i -t --image=alpine test-$RANDOM -- sh
/ # wget -qO- --timeout=2 http://web
wget: download timed out
~~~

可以看到，这次就无法完成了。

-----

## 讲解

在上面的 YAML 中，我们使用`app=web`这样的标签来确定应用网络策略的 Pod 范围。其中的`spec.ingres`字段为空，这样所有到目标 Pod 的流量都会被阻止

如果创建另一个网络策略来让某些 Pod 可以访问我们的目标 Pod，这一策略就会失效。

如果网络策略中包含任意一条规则允许这一流量，就意味着这种流量是有效的，从而忽略掉拦截规则。

## 清理

```sh
kubectl delete deploy web
kubectl delete service web
kubectl delete networkpolicy web-deny-all
```
