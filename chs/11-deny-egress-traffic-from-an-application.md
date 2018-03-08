# 拦截来自某应用的外发流量

## 用例

- 阻止一个应用向 Pod 外发起任何连接。
- 限制单实例的数据库或数据仓库，禁止外发连接。

> **注意**：如果你正在使用的是 GKE，至少需要`1.8.4-gke.0`的 Master 和 Node 才能使用 egress 策略。

## 示例

运行一个标签`app=web`的应用:

~~~sh
kubectl run web --image=nginx --port 80 --expose \
  --labels app=web
~~~

把下面的`foo-deny-egress.yaml`提交到集群运行：

~~~yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: foo-deny-egress
spec:
  podSelector:
    matchLabels:
      app: foo
  policyTypes:
  - Egress
  egress: []
~~~

上面文件的一些说明：

- `podSelector`匹配到标签为`app=foo`的 Pod。
- `policyTypes: ["egress"]`：这一策略负责的是外发通信部分。
- `egress: []`：如果没有设置白名单，那么所有的外发通信都会被禁止。
  - 删除这一字段也会有同样的效果。

~~~sh
kubectl apply -f foo-deny-egress.yaml
networkpolicy "foo-deny-egress" created
~~~

## 测试

用标签`app=foo`运行一个 Pod, 尝试连接`web`服务：

~~~sh
$ kubectl run --rm --restart=Never --image=alpine -i -t -l app=foo test -- ash

/ # wget -qO- --timeout 1 http://web:80/
wget: bad address 'web:80'

/ # wget -qO- --timeout 1 http://www.example.com/
wget: bad address 'www.example.com'
~~~

由于禁止外发流量，导致 Pod 无法发起到`kube-dns`Pod的 DNS 查询，最终使得连接无法进行。

### 允许 DNS 通信

我们对上述规则进行一点修改，允许所有的 DNS 查询(`53/udp`和`53/tcp`)：

~~~yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: foo-deny-egress
spec:
  podSelector:
    matchLabels:
      app: foo
  policyTypes:
  - Egress
  egress:
  # allow DNS resolution
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
~~~

再次运行，会发现 DNS 解析成功，但是仍旧无法访问 Web：

~~~sh
/ # wget --timeout 1 -O- http://web
Connecting to web (10.59.245.232:80)
wget: download timed out

/ # wget --timeout 1 -O- http://www.example.com
Connecting to www.example.com (93.184.216.34:80)
wget: download timed out

/ # ping google.com
PING google.com (74.125.129.101): 56 data bytes
(...)
(ping does not work, hit ctrl+C to terminate)

/ # exit
~~~

Beware that the egress rule above allows Pod to connect not only `kube-dns`,
but any host that serves traffic over port `53`.
目前的规则**不只是**允许 Pod 连接到`kube-dns`，但是，还允许所有 53 的端口通信。

## 清理

~~~sh
kubectl delete deployment,service cache
kubectl delete deployment,service web
kubectl delete networkpolicy foo-deny-egress
~~~
