---
layout: post
---
How to install Kubernetes on Centos7.9  
Software requirements:
```
Kubernetes: 1.20.11
Docker: 19.03.9
CNI: flannel
```

### Requirements
| List | Details | Checkout |
| -- | -- | -- |
| OS　| CentOS 7.9 x86_64  | `cat /etc/centos-release` |
| kernel | 3.10.0 or higher | `uname -r` |
| Swap | No effect on I/O  | `swapoff -a && sed -i 's/.*swap.*/#&/' /etc/fstab`|
| firewall | disable | `systemctl stop firewalld && systemctl disable firewalld` |
| SELinux | Grant access to linux file | `setenforce 0` |
| timezone | Asia/Shanghai | `timedatectl set-timezone Asia/Shanghai` |
| chronyd | delay < 1s when etcd election happen | `vim /etc/chronyc.conf` |
| docker | 19.03.9 or higher | `docker version` |
| kubenetes |  default version 1.20.11 | `kubectl version` |
| CNI | flannel 0.13.0 or higher | `kubectl get pods -n kube-system` |
### network env

`K8S_SVC_CIDR=10.96.0.0/12 `

`K8S_POD_CIDR=10.244.0.0/16`

`K8S_CNI=flannel`


