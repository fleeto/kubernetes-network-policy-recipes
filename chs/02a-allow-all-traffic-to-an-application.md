# 允许所有目标为某个应用的流量


## 用例

在屏蔽所有非白名单范围内的策略生效之后，现在需要设置策略，让命名空间内的所有 Pod 可以访问指定应用。

这一策略提交之后，会清理所有限制到指定 Pod 流量的规则，并允许所有到这一应用的访问。

## 示例

启动一个叫`web`的应用

~~~sh
kubectl run web --image=nginx \
    --labels=app=web --expose --port 80
~~~

生成文件 `web-allow-all.yaml`:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-all
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - {}
```

这一策略的一点说明:

- `namespace: default`：把这一策略发布到`default`命名空间。
- `podSelector`：ingress 规则将会应用到符合标签条件的 Pod 上。
- 只有一条 Ingress 规则，这条规则是空的
  - 空的 Ingress 规则允许当前命名空间内的所有 Pod 进行访问，还可以使用下面的规则来允许所有命名空间的访问
    ~~~yaml
    - from:
        podSelector: {}
        namespaceSelector: {}
    ~~~

将规则应用到集群上：

```sh
$ kubectl apply web-allow-all.yaml
networkpolicy "web-allow-all" created"
```

同时提交前文提到的`web-deny-all`策略。这样就可以验证，`web-allow-all`规则覆盖了`web-deny-all`的行为。

## 测试

    $ kubectl run test-$RANDOM --rm -i -t --image=alpine -- sh
    / # wget -qO- --timeout=2 http://web
    <!DOCTYPE html>
    <html><head>
    ...

成功进行通信。

### 清理

```sh
kubectl delete deployment,service web
kubectl delete networkpolicy web-allow-all web-deny-all
```
