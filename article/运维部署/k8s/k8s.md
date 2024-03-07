> 官方手册 https://kubernetes.io/zh-cn/docs/reference/kubectl/



# 1. 安装k8s集群

## 1.1通过kubeadm安装

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

> 生产环境k8s集群建议使用二进制文件安装



### 1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区

### 2. 准备环境

| 角色   | IP           |
| ------ | ------------ |
| master | 192.168.1.11 |
| node1  | 192.168.1.12 |
| node2  | 192.168.1.13 |

```shell
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 在master添加hosts
cat >> /etc/hosts << EOF
192.168.44.146 k8smaster
192.168.44.145 k8snode1
192.168.44.144 k8snode2
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com
```

### 3. 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

#### 3.1 安装Docker

```shell
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
$ yum -y install docker-ce-18.06.1.ce-3.el7
$ systemctl enable docker && systemctl start docker
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
$ cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

#### 3.2 添加阿里云YUM软件源

```shell
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 3.3 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```shell
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
$ systemctl enable kubelet
```

### 4. 部署Kubernetes Master

在192.168.31.61（Master）执行。

```shell
$ kubeadm init \
  --apiserver-advertise-address=192.168.44.146 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

使用kubectl工具：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
```

### 5. 加入Kubernetes Node

在192.168.1.12/13（Node）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```shell
$ kubeadm join 192.168.1.11:6443 --token esce21.q6hetwm8si29qxwn \
    --discovery-token-ca-cert-hash sha256:00603a05805807501d7181c3d60b478788408cfe6cedefedb1f97569708be9c5
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```shell
kubeadm token create --print-join-command
```

### 6. 部署CNI网络插件

```shell
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

默认镜像地址无法访问，sed命令修改为docker hub镜像仓库。

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
kube-flannel-ds-amd64-2pc95   1/1     Running   0          72s
```

### 7. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```shell
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```

访问地址：http://NodeIP:Port  





## 1.2 通过二进制文件安装



todo



## 2. yaml文件

k8s 集群中对资源管理和资源对象编排部署都可以通过声明样式(YAML)文件来解决，也就是可以把需要对资源对象操作编辑到 YAML 格式文件中，我们把这种文件叫做资源清单文件.



### 资源编排文件格式语法

以service.yaml 文件语法举例

```yaml
apiVersion: v1     # 指定api版本，此值必须在kubectl api-versions中 
kind: Service     # 指定创建资源的角色/类型 
metadata:     # 资源的元数据/属性
  name: demo     # 资源的名字，在同一个namespace中必须唯一
  namespace: default     # 部署在哪个namespace中。不指定时默认为default命名空间
  labels:         # 设定资源的标签
  - app: demo
  annotations:  # 自定义注解属性列表
  - name: string
spec:     # 资源规范字段
  type: ClusterIP     # service的类型，指定service的访问方式，默认ClusterIP。
      # ClusterIP类型：虚拟的服务ip地址，用于k8s集群内部的pod访问，在Node上kube-porxy通过设置的iptables规则进行转发
      # NodePort类型：使用宿主机端口，能够访问各个Node的外部客户端通过Node的IP和端口就能访问服务器
      # LoadBalancer类型：使用外部负载均衡器完成到服务器的负载分发，需要在spec.status.loadBalancer字段指定外部负载均衡服务器的IP，并同时定义nodePort和clusterIP用于公有云环境。
  clusterIP: string        #虚拟服务IP地址，当type=ClusterIP时，如不指定，则系统会自动进行分配，也可以手动指定。当type=loadBalancer，需要指定
  sessionAffinity: string    #是否支持session，可选值为ClietIP，默认值为空。ClientIP表示将同一个客户端(根据客户端IP地址决定)的访问请求都转发到同一个后端Pod
  ports:
    - port: 8080     # 服务监听的端口号
      targetPort: 8080     # 容器暴露的端口
      nodePort: int        # 当type=NodePort时，指定映射到物理机的端口号
      protocol: TCP     # 端口协议，支持TCP或UDP，默认TCP
      name: http     # 端口名称
  selector:     # 选择器。选择具有指定label标签的pod作为管理范围
    app: demo
status:    # 当type=LoadBalancer时，设置外部负载均衡的地址，用于公有云环境    
  loadBalancer:    # 外部负载均衡器    
    ingress:
      ip: string    # 外部负载均衡器的IP地址
      hostname: string    # 外部负载均衡器的主机名
```



- apiVersion, 使用的api版本, `kubectl api-versions`查看所有支持的版本
  
  只要记住6个常用的apiversion一般就够用了。

  - **v1**： Kubernetes API的稳定版本，包含很多核心对象：pod、service等。
  - **apps/v1**： 包含一些通用的应用层的api组合，如：Deployments, RollingUpdates, and ReplicaSets。
  - **batch/v1**： 包含与批处理和类似作业的任务相关的对象，如：job、cronjob。
  - **autoscaling/v1**： 允许根据不同的资源使用指标自动调整容器。
  - **networking.k8s.io/v1**： 用于Ingress。
  - **rbac.authorization.k8s.io/v1**：用于RBAC。
  - [kubernetes.io apiVersion详细说明](https://link.juejin.cn?target=https%3A%2F%2Fkubernetes.io%2Fdocs%2Freference%2Fusing-api%2F)



- kind: 指定这个资源对象的类型，如 pod、deployment、statefulset、job、cronjob
- metadata: 资源的元数据
- spec: 资源的规格配置, 是重要/主要的配置项，支持的子项非常多，根据资源对象的不同，子项会有不同的配置。

- port
  - **port**：port是k8s集群**内部访问service的端口**，即通过clusterIP: port可以访问到某个service
  - nodePort**：nodePort是**外部访问k8s集群中service的端口**，通过nodeIP: nodePort可以从外部访问到某个service。**
  - targetPort**：targetPort是**pod的端口**，从port和nodePort来的流量经过kube-proxy流入到后端pod的targetPort上，最后进入容器。**
  - containerPort**：containerPort是**pod内部容器的端口**，targetPort映射到containerPort。



**一个pod的yaml：**

```yaml
apiVersion: v1 #必选，版本号，例如v1
kind: Pod #必选，Pod 
metadata: #必选，元数据 
  name: nginx #必选，Pod名称 
  labels: #自定义标签 
     app: nginx #自定义标签名字 
spec: #必选，Pod中容器的详细定义 
     containers: #必选，Pod中容器列表，一个pod里会有多个容器 
        - name: nginx #必选，容器名称 
          image: nginx #必选，容器的镜像名称 
          imagePullPolicy: IfNotPresent # [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像 
          ports: #需要暴露的端口库号列表 
          - containerPort: 80 #容器需要监听的端口号 
     restartPolicy: Always # [Always | Never | OnFailure]#Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod 
```

**一个ingress的yaml** 

```yaml
apiVersion: extensions/v1beta1         # 创建该对象所使用的 Kubernetes API 的版本     
kind: Ingress         # 想要创建的对象的类别，这里为Ingress
metadata:
  name: showdoc        # 资源名字，同一个namespace中必须唯一
  namespace: op     # 定义资源所在命名空间
  annotations:         # 自定义注解
    kubernetes.io/ingress.class: nginx        # 声明使用的ingress控制器
spec:
  rules:
  - host: showdoc.example.cn     # 服务的域名
    http:
      paths:
      - path: /      # 路由路径
        backend:     # 后端Service
          serviceName: showdoc        # 对应Service的名字
          servicePort: 80           # 对应Service的端口
```





### 获取yaml文件方式



1. 通过create命令来生成yaml文件,  `kubectl create deployment ng --image=nginx -o yaml --dry-run=client`
   - `-o yaml`打印过程yaml内容, 
   - `--dry-run=client`, 不会执行命令

```shell
[root@master ~]# kubectl create deployment ng1 --image=nginx -o yaml --dry-run=client
W0307 23:52:54.624253   95844 helpers.go:535] --dry-run is deprecated and can be replaced with --dry-run=client.
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: ng1
  name: ng1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ng1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ng1
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```



2. 通过get命令导出运行中的资源的yaml文件

```shell
[root@master ~]# kubectl get deployments nginx -o=yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
....
```



3. `explain`查看yaml的所有字段、默认值和示例的详细信息,

我现在要知道deployment对象的spec属性的还有哪些可用的字段。

```shell
[root@master ~]# kubectl explain deployment.spec
KIND:     Deployment
VERSION:  apps/v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the Deployment.

     DeploymentSpec is the specification of the desired behavior of the
     Deployment.

FIELDS:
   minReadySeconds      <integer>
     Minimum number of seconds for which a newly created pod should be ready
     without any of its container crashing, for it to be considered available.
     Defaults to 0 (pod will be considered available as soon as it is ready)

   paused       <boolean>
     Indicates that the deployment is paused.

   progressDeadlineSeconds      <integer>
     The maximum time in seconds for a deployment to make progress before it is
     considered to be failed. The deployment controller will continue to process
     failed deployments and a condition with a ProgressDeadlineExceeded reason
     will be surfaced in the deployment status. Note that progress will not be
     estimated during the time a deployment is paused. Defaults to 600s.

   replicas     <integer>
     Number of desired pods. This is a pointer to distinguish between explicit
     zero and not specified. Defaults to 1.

   revisionHistoryLimit <integer>
     The number of old ReplicaSets to retain to allow rollback. This is a
     pointer to distinguish between explicit zero and not specified. Defaults to
     10.

   selector     <Object> -required-
     Label selector for pods. Existing ReplicaSets whose pods are selected by
     this will be the ones affected by this deployment. It must match the pod
     template's labels.

   strategy     <Object>
     The deployment strategy to use to replace existing pods with new ones.

   template     <Object> -required-
     Template describes the pods that will be created.
```







# 2. kubernetes 集群命令行工具 kubectl

kubectl 是 Kubernetes 集群的命令行工具，通过 kubectl 能够对集群本身进行管理，并能 够在集群上进行容器化应用的安装部署。

kubectl 命令的语法

```shell
kubectl [command] [TYPE] [NAME] [flags]
```

其中 `command`、`TYPE`、`NAME` 和 `flags` 分别是：

- `command`：指定要对一个或多个资源执行的操作，例如 `create`、`get`、`describe`、`delete`。

- `TYPE`：指定资源类型。资源类型不区分大小写， 可以指定单数、复数或缩写形式。例如，以下命令输出相同的结果：

  ```shell
  kubectl get pod pod1
  kubectl get pods pod1
  kubectl get po pod1
  ```

- `NAME`：指定资源的名称。名称区分大小写。 如果省略名称，则显示所有资源的详细信息。例如：`kubectl get pods`。

  在对多个资源执行操作时，你可以按类型和名称指定每个资源，或指定一个或多个文件：

  - 要按类型和名称指定资源：

  - 要对所有类型相同的资源进行分组，请执行以下操作：`TYPE1 name1 name2 name<#>`。
    例子：`kubectl get pod example-pod1 example-pod2`

  - 分别指定多个资源类型：`TYPE1/name1 TYPE1/name2 TYPE2/name3 TYPE<#>/name<#>`。
    例子：`kubectl get pod/example-pod1 replicationcontroller/example-rc1`

  - 用一个或多个文件指定资源：`-f file1 -f file2 -f file<#>`

  - 使用 YAML 而不是 JSON， 因为 YAML 对用户更友好, 特别是对于配置文件。
    例子：`kubectl get -f ./pod.yaml`

- `flags`： 指定可选的参数。例如，可以使用 `-s` 或 `--server` 参数指定 Kubernetes API 服务器的地址和端口。

- `kubectl --help` 查看详细说明

































# 附录



## 资源缩写

```shell
[root@master ~]# kubectl api-resources 
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
bindings                                                                      true         Binding
componentstatuses                 cs                                          false        ComponentStatus
configmaps                        cm                                          true         ConfigMap
endpoints                         ep                                          true         Endpoints
events                            ev                                          true         Event
limitranges                       limits                                      true         LimitRange
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumeclaims            pvc                                         true         PersistentVolumeClaim
persistentvolumes                 pv                                          false        PersistentVolume
pods                              po                                          true         Pod
podtemplates                                                                  true         PodTemplate
replicationcontrollers            rc                                          true         ReplicationController
resourcequotas                    quota                                       true         ResourceQuota
secrets                                                                       true         Secret
serviceaccounts                   sa                                          true         ServiceAccount
services                          svc                                         true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
controllerrevisions                            apps                           true         ControllerRevision
daemonsets                        ds           apps                           true         DaemonSet
deployments                       deploy       apps                           true         Deployment
replicasets                       rs           apps                           true         ReplicaSet
statefulsets                      sts          apps                           true         StatefulSet
tokenreviews                                   authentication.k8s.io          false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io           true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling                    true         HorizontalPodAutoscaler
cronjobs                          cj           batch                          true         CronJob
jobs                                           batch                          true         Job
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
leases                                         coordination.k8s.io            true         Lease
endpointslices                                 discovery.k8s.io               true         EndpointSlice
events                            ev           events.k8s.io                  true         Event
ingresses                         ing          extensions                     true         Ingress
ingressclasses                                 networking.k8s.io              false        IngressClass
ingresses                         ing          networking.k8s.io              true         Ingress
networkpolicies                   netpol       networking.k8s.io              true         NetworkPolicy
runtimeclasses                                 node.k8s.io                    false        RuntimeClass
poddisruptionbudgets              pdb          policy                         true         PodDisruptionBudget
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io      true         RoleBinding
roles                                          rbac.authorization.k8s.io      true         Role
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
csidrivers                                     storage.k8s.io                 false        CSIDriver
csinodes                                       storage.k8s.io                 false        CSINode
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment
```





## 设置命令函自动补齐

```shell
yum install -y bash-completion -y

chmod +x /usr/share/bash-completion/bash_completion
/usr/share/bash-completion/bash_completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
```

这时补全已经可以用了， kubectl desc<tab> . 但必须使用kubectl的全称，使用别名不行。
设置登录自动加载：

```shell
echo "source <(kubectl completion bash)" >> /etc/bashrc
```






![Kubernetes-Cheat-Sheet](/Users/bearomac/gitbook/article/运维部署/img/Kubernetes-Cheat-Sheet.png)



## 搭建本地k8s环境

## minikube搭建一个单节点的kubernetes集群环境



国内

```shell
minikube start --image-mirror-country='cn' --registry-mirror=https://registry.docker-cn.com  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

minikube dashboard
```





## Multipass和k3s来搭建一个多节点的kubernetes集群环境

k3s 是一个轻量级的Kubernetes发行版，它是 Rancher Labs 推出的一个开源项目，

旨在简化Kubernetes的安装和维护，同时它还是CNCF认证的Kubernetes发行版。





## 在线实验环境 

Killercoda
Play-With-K8s
