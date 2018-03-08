# 用多个选择器选择允许通过的流量

网络策略功能允许用户定义多个 Pod 选择器来选择允许通过的范围。

## 用例

- 创建一个联合策略，列出允许访问某一应用的微服务。

## 示例

在集群里运行一个 Redis：

~~~sh
kubectl run db --image=redis:4 --port 6379 --expose \
        --labels app=bookstore,role=db
~~~


假设有多个服务服务需要访问这个 Redis：

| 服务    | 标签 |
|------------|--------|
| `search`   | `app=bookstore` `role=search` |
| `api`      | `app=bookstore` `role=api` |
| `catalog`  | `app=inventory` `role=web` |

下面的网络策略设置只允许这几个服务访问 Redis，保存为 `redis-allow-services.yaml`，并提交到集群：

~~~yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: redis-allow-services
spec:
  podSelector:
    matchLabels:
      app: bookstore
      role: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: bookstore
          role: search
    - podSelector:
            matchLabels:
              app: bookstore
              role: api
    - podSelector:
            matchLabels:
              app: inventory
              role: web
~~~

~~~sh
$ kubectl apply -f redis-allow-services.yaml
networkpolicy "redis-allow-services" created
~~~

注意：

- 在`spec.ingress.from`中的多个规则是`OR`关系
- 这些选择器会成为一个白名单

## 测试

按照上面的`catalog`规则进行部署：

~~~sh
$ kubectl run test-$RANDOM --labels=app=inventory,role=web --rm -i -t --image=alpine -- sh

/ # nc -v -w 2 db 6379
db (10.59.242.200:6379) open

(works)
~~~

如果 Pod 不符合条件则无法完成：

~~~sh
$ kubectl run test-$RANDOM --labels=app=other --rm -i -t --image=alpine -- sh

/ # nc -v -w 2 db 6379
nc: db (10.59.252.83:6379): Operation timed out

(traffic blocked)
~~~

### 清理

~~~sh
kubectl delete deployment db
kubectl delete service db
kubectl delete networkpolicy redis-allow-services
~~~