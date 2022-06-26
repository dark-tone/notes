Deployment，与 ReplicaSet，以及 Pod的关系：

<img src="https://raw.githubusercontent.com/dark-tone/notes/main/k8s/imgs/2.webp" style="width: 650px;">

设置副本指令
```
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```


- 金丝雀部署：优先发布一台或少量机器升级，等验证无误后再更新其他机器。优点是用户影响范围小，不足之处是要额外控制如何做自动更新。
- 蓝绿部署：2组机器，蓝代表当前的V1版本，绿代表已经升级完成的V2版本。通过LB将流量全部导入V2完成升级部署。优点是切换快速，缺点是影响全部用户。