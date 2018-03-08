# 允许所有来自于某命名空间的流量

这一策略和之前的允许全部命名空间的策略很相似，但是只选择部分命名空间。

## 用例

- 限制只能由生产环境的应用发起对生产数据库的访问。

- 部署监控工具，用于从当前命名空间中的应用获取指标。

![Diagram of ALLOW all traffic from a namespace policy](img/6.gif)

## 示例

在`default`命名空间中运行一个 Web Server：

~~~sh
kubectl run web --image=nginx \
    --labels=app=web --expose --port 80
~~~

假设有三个命名空间：

- `default`: 集群的缺省命名空间，在其中部署 API。
- `prod`: 运行其他的工作负载，带有标签`purpose=prod`。
- `dev`: 开发测试环境，标签为`purpose=testing`.

创建`prod`和`dev`命名空间：

~~~sh
kubectl create namespace dev
kubectl label namespace/dev purpose=testing
~~~

~~~sh
kubectl create namespace prod
kubectl label namespace/prod purpose=production
~~~

下面的文件限制到`app: web` Pod 的访问：只允许来自`purpose=production`命名空间，文件命名为`web-allow-prod.yaml`，并提交给集群：

~~~yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-prod
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: production
~~~

~~~sh
$ kubectl apply -f web-allow-prod.yaml
networkpolicy "web-allow-prod" created
~~~

## 测试

从`dev`发起访问，会发现被屏蔽：

~~~sh
$ kubectl run test-$RANDOM --namespace=dev --rm -i -t --image=alpine -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -qO- --timeout=2 http://web.default
wget: download timed out

(traffic blocked)
~~~

从`prod`命名空间访问，

~~~sh
$ kubectl run test-$RANDOM --namespace=prod --rm -i -t --image=alpine -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -qO- --timeout=2 http://web.default
<!DOCTYPE html>
<html>
<head>
...
(traffic allowed)
~~~

## 清理

~~~sh
kubectl delete networkpolicy web-allow-prod
kubectl delete deployment web
kubectl delete service web
kubectl delete namespace {prod,dev}
~~~