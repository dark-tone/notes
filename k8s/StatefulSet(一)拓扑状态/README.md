# 与Deployment区别
Deployment部署无状态应用。
1. pod之间没有顺序
2. 所有pod共享存储
3. pod名字包含随机数字
4. service都有ClusterIP,可以负载均衡

StatefulSet部署有状态应用。
1. 部署、扩展、更新、删除都要有顺序
2. 每个pod都有自己存储，所以都用volumeClaimTemplates，为每个pod都生成一个自己的存储，保存自己的状态
3. pod名字始终是固定的
4. service没有ClusterIP，是headlessservice，所以无法负载均衡，返回的都是pod名，所以pod名字都必须固定，StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod副本创建了一个DNS域名:$(podname).(headless server name).namespace.svc.cluster.local

# Service访问方式
## VirtualIP
比如：当我访问 10.0.23.1 这个 Service 的 IP 地址时，10.0.23.1 其实就是一个 VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上。

## DNS
比如：这时候，只要我访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录，就可以访问到名叫 my-svc 的 Service 所代理的某一个 Pod。

具体又可细分为两种方式：
- Normal Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，正是 my-svc 这个 Service 的 VIP，后面的流程就跟 VIP 方式一致了。
- Headless Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址。**可以看到，这里的区别在于，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。**

# yaml示例
**headless示例**
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```
所谓的 Headless Service，其实仍是一个标准 Service 的 YAML 文件。只不过，它的 clusterIP 字段的值是：None。

**StatefulSet示例：**
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```
通过serviceName=nginx匹配headless service。

通过 Headless Service 的方式，StatefulSet 为每个 Pod 创建了一个固定并且稳定的 DNS 记录，来作为它的访问入口。这种严格的对应规则，就保证了 Pod 网络标识的稳定性。
> 创建statefulset必须要先创建一个headless的service

对于“有状态应用”实例的访问，你必须使用 DNS 记录或者 hostname 的方式，而绝不应该直接访问这些 Pod 的 IP 地址。

# 参考资料
[Deployment和Statefulset区别](https://blog.csdn.net/huiqiwei321/article/details/108143085)