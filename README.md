# kubernetes-item
k8s learn

k8s 学习过程


- 每台服务器都需要安装Docker服务


```yaml
// 准备四台服务器
10.200.141.156 harbor

10.200.141.154 master
10.200.141.157 node
10.200.141.158 node

```

# harbor


安装
```yaml
wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.0.tgz
tar -zxvf harbor-offline-installer-v1.8.0.tgz
cd harbor/
vi harbor.yml
  hostname: 10.200.141.156

// 启动服务
./install.sh
```

访问服务
```
harbor：docker私库
http://10.200.141.156
user User12345
```
**我在安装k8s前，已经把需要的镜像文件都放在私库中了，可以登录查看**

![Image text](https://github.com/woxiaobao/kubernetes-item/blob/master/img-folder/harbor.png)


### k8s yum安装过程
kubernetes  v1.5.2


# master节点安装etcd

安装
```
  yum install etcd -y
```

修改配置
```
 vim /etc/etcd/etcd.conf
 
ETCD_LISTEN_CLIENT_URLS=“http://0.0.0.0:2379”
ETCD_ADVERTISE_CLIENT_URLS=“http://10.200.141.154:2379”
```

启动服务，设置为开机启动
```
systemctl start etcd.service
systemctl enable etcd.service
```


测试etcd

```
etcdctl set testdir/key0 1
etcdctl get testdir/key0
```

检查集群健康状态
```
etcdctl -C http://10.200.141.154:2379 cluster-health
```




3、master 节点安装kubernetes
```
yum install kubernetes-master.x86_64 -y
```
```
vim  /etc/kubernetes/apiserver

KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_ETCD_SERVERS="--etcd-servers=http://10.200.141.154:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"

```

```
vim /etc/kubernetes/config

KUBE_MASTER="--master=http://10.200.141.154:8080"
```

￼
```
systemctl enable kube-apiserver.service
systemctl restart kube-apiserver.service
systemctl enable kube-controller-manager.service
systemctl restart kube-controller-manager.service
systemctl enable kube-scheduler.service
systemctl restart kube-scheduler.service
```

查看组件的健康状态
```
kubectl get componentstatus
```

# node节点安装kubernetes
```
yum install kubernetes-node.x86_64 -y
```
```
￼vim /etc/kubernetes/config

KUBE_MASTER="--master=http://10.200.141.154:8080"
```
修改kubelet配置
```
vim /etc/kubernetes/kubelet

KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME="--hostname-override=10.200.141.157"
KUBELET_API_SERVER="--api-servers=http://10.200.141.154:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=10.200.141.156/item/pod-infrastructure:latest"
```



```
systemctl enable kubelet.service
systemctl restart kubelet.service
systemctl enable kube-proxy.service
systemctl restart kube-proxy.service
```

master节点查看node节点
```
kubectl get nodes
```

# 所有节点都安装上flannel网络
```
yum install flannel -y
```
```
 vim /etc/sysconfig/flanneld

FLANNEL_ETCD_ENDPOINTS="http://10.200.141.154:2379"
```
master节点执行
```
etcdctl mk /atomic.io/netword/config '{ "Network":"172.16.0.0/16" }'
```

master节点
```
systemctl enable flanneld.service
systemctl start flanneld.service
systemctl restart docker
systemctl restart kube-apiserver.service
systemctl restart kube-controller-manager.service
systemctl restart kube-scheduler.service

```

node节点
```
systemctl enable flanneld.service
systemctl start flanneld.service
systemctl restart docker
systemctl restart kubelet.service
systemctl restart kube-proxy.service
```

Docker设定为私库
```
vim /etc/sysconfig/docker
OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false --registry-mirror=https://registry.docker-cn.com --insecure-registry=10.200.141.156'
systemctl restart docker
```

### 创建一个pod
```
kubectl create -f k8s_pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels: 
    app: web
spec:
  containers:
    - name: nginx
      image: 10.200.141.156/item/nginx:v1
      ports:
        - containerPort: 80
```



### 查看pod情况详细信息
```
kubectl get pods -o wide
kubectl describe pod nginx
```

根据rc名称编辑文件
```
kubectl edit rc myweb
```

可以通过scale直接修改副本数量，修改rc名称为：myweb的副本数量为3
```
kubectl scale replicationcontroller myweb --replicas=3
或 kubectl scale rc myweb --replicas=3
```


滚动升级 (每5秒升级一个副本)
```
kubectl rolling-update myweb -f k8s-rcv2.yaml --update-period=5s
```


进入容器中
```
kubectl exec -it pod-name /bin/bash
```

查看yaml   文件中的写法
```
kubectl explain svc.spec
kubectl explain pod.spec.containers
```

 不开启将导致service无法连通不同node
//开放中转端
```
iptables -P FORWARD ACCEPT 
```
- https://blog.csdn.net/alanmomo/article/details/86003882


查看所有资源
```
kubectl get all 
```

# deployment
Rc的缺点：
Rc: edit修改rc配置，只有启动新的pod的时候才能生效
Rc滚动升级后，pod的标签页升级了，导致service关联不上

rs 有rc的95%功能，rs是由deployment自动创建 

查看名称为nginx-depl的deployment历史版本
```
kubectl rollout history deploment nginx-depl
```

回滚到历史版本1
```
kubectl rollout undo deployment nginx-depl —to-revision=1
```

直接创建一个deployment，name为nginx-depl,没有指定标签也为nginx-depl
```
kubectl run  nginx-depl --image=10.200.141.156/item/nginx:v1 --replicas=4 --record
```

直接根据deployment名称为nginx-depl,pod标签为Nginx修改镜像 增加--record 会在版本
历史记录中记录下这条命令
```
kubectl set image deployment nginx-depl nginx=10.200.141.156/item/nginx:v1 --record
```

命令创建svc
```
kubectl expose deployment nginx-depl --port=80 --type=NodePort
```



# Horizontal Pod Autoscaler

实现HPA   Horizontal Pod Autoscaler
Pod的水平自动扩展

# heapster
1、搭建k8s监控

创建一个目录，上传5个yaml文件
2、上传三个docker镜像到私有仓库
3、创建当前目录下所有的yaml
```
kubectl create -f .
```
4、验证
```
kubectl get all --namespace=kube-system
kubectl describe pod podName -n kube-system
```

# 搭建 dashboard
1、上传dashboard镜像到私有仓库
2、根据yaml创建k8s资源
```
kubectl create -f dashboard.yaml
kubectl create -f dashboard-svc.yaml
kubectl get all --namespace=kube-system
```
3、访问dashboard
```
http://10.200.141.154:8080/ui/
```
![Image text](https://github.com/woxiaobao/kubernetes-item/blob/master/img-folder/dashboard.png)

创建HPA，pod最小数量1，pod最大数量8，cpu使用率百分比作为触发条件
```
kubectl autoscale deployment nginx-depl --max=8 --min=1 --cpu-percent=5
```
压测请求命令
```
 while true; do curl 172.16.75.3 ; done
```


# dns
dns的功能就是把svc的name解析成vip
通过名称访问服务
修改各个node节点上的
```
vim /etc/kubernetes/kubelet
KUBELET_ARGS="--cluster_dns=10.254.230.254 --cluster_domain=cluster.local"
```
实例：
http://10.200.141.154:8080/api/v1/proxy/namespaces/baolin/pods/nginx

# volume持久化
pv持久化存储   对接来源 打标签
pv是全局不区分namespace

pvc持久化存储，分配 只负责用 选择标签
pvc是区分namespace

# nfs存储方式安装
1、在master节点安装nfs服务端

```
yum install nfs-utils.x86_64 -y
vim /etc/exports
//下面的网段不能错
/data 10.200.141.0/24(rw,async,no_root_squash,no_all_squash)
mkdir /data/web -p
systemctl restart rpcbind
systemctl restart nfs
```

2、每一个node节点都需要安装nfs客户端
```
yum install nfs-utils.x86_64 -y
```
测试nfs服务器挂载目录
```
showmount -e 10.200.141.154
```






# 学习方向
- service 4层
- ingress 7层负载均衡
- k8s的监控 prometheus监控
- k8s的日志 EFK
- k8s的包管理器 helm
- k8s的网络插件 flannel, calico
- k8s的存储，clusterFS , ceph