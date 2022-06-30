Kubernetes 之所以需要 Service，一方面是因为 Pod 的 IP 不是固定的，另一方面则是因为一组 Pod 实例之间总会有负载均衡的需求。

yaml例子：
```
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

Service 是由 kube-proxy 组件，加上 iptables 来共同实现的。

# 类型
## ClusterIP
默认类型，自动分配一个仅Cluster内部可以访问的虚拟IP

## NodePort
在ClusterIP基础上为Service在每台机器上绑定一个端口，这样就可以通过 **<任何一台宿主机的IP地址>:<端口>** 进行访问。

## LoadBalancer
在NodePort的基础上，借助Cloud Provider创建一个外部负载均衡器，并将请求转发到NodePort

## ExternalName
把集群外部的服务引入到集群内部来，在集群内部直接使用。没有任何类型代理被创建，这只有 Kubernetes 1.7或更高版本的kube-dns才支持。

# 参考资料
[一文讲透K8s的Service概念](https://baijiahao.baidu.com/s?id=1681303264708121640&wfr=spider&for=pc)