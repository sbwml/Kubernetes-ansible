# Kubernetes-ansible

## ansible部署Kubernetes

系统可采用`Ubuntu 16.x`与`CentOS 7.x`
本次安裝的版本：
> * Kubernetes v1.10.0 (HA高可用)
> * CNI v0.6.0
> * Etcd v3.1.13
> * Calico v3.0.4
> * Docker CE latest version(18.03)

**管理组件采用`staticPod`或者`daemonSet`形式跑的,仅供用于学习和实验**

安装过程是参考的[Kubernetes v1.10.x HA全手动苦工安装教学](https://zhangguanzhang.github.io/2018/05/05/Kubernetes_install/)

**下面是我的配置,电脑配置低就一个Node节点**

| IP    | Hostname   |  CPU  |   Memory | 
| :----- |  :----:  | :----:  |  :----:  |
| 192.168.126.6 |K8S-M1|  1   |   2G    |
| 192.168.126.7 |K8S-M2|  1   |   2G    |
| 192.168.126.8 |K8S-M3|  1   |   2G    |
| 192.168.126.9 |K8S-N1|  1   |   2G    |

# 使用前提和注意事项（所有主机）
> * 关闭selinux和disbled防火墙(确保getenforce的值是Disabled配置文件改了后应该重启)
> * swap(/etc/fstab也关闭)
> * 设置ntp同步时间
> * 安装epel源和openssl和expect
> * 设置各台主机名(参照我那样,分发hosts看下面使用)
> * 每台主机端口和密码最好一致(不一致最好懂点ansible)
> * 设置内核转发(参照脚本里的一部分设置)
> * 安装年份命名的版本的Docker CE(Centos7.x建议先yum update后再安装)

以上一部分可以使用我写的env_set.sh脚本(仅适用于Centos)

# 使用(在master1的主机上使用且master1安装了ansible)
**1 git clone**
```
git clone https://github.com/zhangguanzhang/Kubernetes-ansible.git
cd Kubernetes-ansible
```
`github`文件大小限制推送,`kubectl`和`kubelet`大小太大我上传百度云了,自行下载[download](https://pan.baidu.com/s/1v7uN4ht-7qvA1uk9ZMmuMA),文件下载后位置存放参考`FileTree.txt`里的结构

**2 配置脚本属性**

 * 修改当前目录ansible的`hosts`分组成员文件,填写otherMaster和Node下面填写各成员的ip地址,localhost那部分和分组名别动



 * 修改`group_vars/all.yml`里面的参数
 1. ansible_ssh_pass为ssh密码(如果每台主机密码不一致请注释掉`all.yml`里的`ansible_ssh_pass`按照的`hosts`里的注释那样写上每台主机的密码）
 2. TOKEN可以使用`head -c 32 /dev/urandom | base64`生成替换
 3. TOKEN_ID可以使用`openssl rand 3 -hex`生成
 4. TOKEN_SECRET使用`openssl rand 8 -hex`
 5. VIP为高可用HA的虚ip,NETMASK为VIP的掩码
 6. INTERFACE_NAME为各机器的ip所在网卡名字Centos可能是ens33,看情况修改
 7. 其余的参数按需修改,不熟悉最好别乱改
----------

**3 手动分发hosts文件**
修改本机`/etc/hosts`文件改成这样的格式
```
...
192.168.123.6 k8s-m1
192.168.123.7 k8s-m2
192.168.123.8 k8s-m3
192.168.123.9 k8s-n1
```
然后使用下面命令来分发hosts文件(如果每台主机密码不一致确保ansible的hosts文件里写了每台主机的ansible_ssh密码和端口下再使用此命令分发hosts文件)
```
ansible all -m copy -a 'src=/etc/hosts dest=/etc/hosts'
```
**4 开始运行安装(虚拟机的话建议现在可以关机做个快照以防万一)**

 * 因为有些镜像需要拉取,所以是分成三部,step1是master的管理组件,step2是TLS+NODE,step3是Dashboard+Heapster(不需要Heapster的话注释掉roles/KubernetesExtraAddons/tasks/main.yml里的相关部分)
 1. ansible-playbook -i hosts step1.yml后等待以下输出
```
$ watch netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      10344/kubelet
tcp        0      0 127.0.0.1:10251         0.0.0.0:*               LISTEN      11324/kube-schedule
tcp        0      0 0.0.0.0:6443            0.0.0.0:*               LISTEN      11416/haproxy
tcp        0      0 127.0.0.1:10252         0.0.0.0:*               LISTEN      11235/kube-controll
tcp        0      0 0.0.0.0:9090            0.0.0.0:*               LISTEN      11416/haproxy
tcp6       0      0 :::2379                 :::*                    LISTEN      10479/etcd
tcp6       0      0 :::2380                 :::*                    LISTEN      10479/etcd
tcp6       0      0 :::10255                :::*                    LISTEN      10344/kubelet
tcp6       0      0 :::5443                 :::*                    LISTEN      11295/kube-apiserve
$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}

$ kubectl get node
NAME      STATUS     ROLES     AGE       VERSION
k8s-m1    NotReady   master    52s       v1.10.0
k8s-m2    NotReady   master    51s       v1.10.0
k8s-m3    NotReady   master    50s       v1.10.0

$ kubectl -n kube-system get po
NAME                             READY     STATUS    RESTARTS   AGE
etcd-k8s-m1                      1/1       Running   0          7m
etcd-k8s-m2                      1/1       Running   0          8m
etcd-k8s-m3                      1/1       Running   0          7m
haproxy-k8s-m1                   1/1       Running   0          7m
haproxy-k8s-m2                   1/1       Running   0          8m
haproxy-k8s-m3                   1/1       Running   0          8m
keepalived-k8s-m1                1/1       Running   0          8m
keepalived-k8s-m2                1/1       Running   0          7m
keepalived-k8s-m3                1/1       Running   0          7m
kube-apiserver-k8s-m1            1/1       Running   0          7m
kube-apiserver-k8s-m2            1/1       Running   0          6m
kube-apiserver-k8s-m3            1/1       Running   0          7m
kube-controller-manager-k8s-m1   1/1       Running   0          8m
kube-controller-manager-k8s-m2   1/1       Running   0          8m
kube-controller-manager-k8s-m3   1/1       Running   0          8m
kube-scheduler-k8s-m1            1/1       Running   0          8m
kube-scheduler-k8s-m2            1/1       Running   0          8m
kube-scheduler-k8s-m3            1/1       Running   0          8m
```
 2. 上面输出一致即可运行`step2.yml`
 3. step2.yml运行完后通过下面命令查看如下输出确保dns的3个pod即可运行step3.yml
```
$ kubectl -n kube-system get po -l k8s-app=kube-dns
kubectl -n kube-system get po -l k8s-app=kube-dns
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-654684d656-j8xzx   3/3       Running   0          10m

```
 4. 访问地址会在master1的家目录生成对应的使用指导的txt文件,获取Dashboard的token脚本在家目录下




