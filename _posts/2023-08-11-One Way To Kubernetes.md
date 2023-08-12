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

### network env

`K8S_SVC_CIDR=10.96.0.0/12 `

`K8S_POD_CIDR=10.244.0.0/16`

`K8S_CNI=flannel`


### basic env

---
work,log,data dir
```
mkdir -p /iflytek/{servers,logs,data,upload}
```

kernel and network config
```
$ vim /etc/sysctl.conf

# kernel config
fs.file-max = 1048576
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 5
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2 
vm.max_map_count = 262144

# active network config
sysctl -w vm.max_map_count=262144

```

### ulimit config

```
$ vim /etc/security/limits.conf
# add
* soft memlock unlimited
* hard memlock unlimited
* soft nproc 102400
* hard nproc 102400
* soft nofile 1048576
* hard nofile 1048576

### prepare basic env

---

### install ansible 
```

---
1. description

| List | Details | Checkout |
| -- | -- | -- |
| OS　| CentOS 7.9 x86_64  | `cat /etc/centos-release` |
| ansible | 2.9.27 or higher | deploy |
| kernel | 3.10.0 or higher | `uname -r` |
| Swap | No effect on I/O  | `swapoff -a && sed -i 's/.*swap.*/#&/' /etc/fstab`|
| firewall | disable | `systemctl stop firewalld && systemctl disable firewalld` |
| SELinux | Grant access to linux file | `setenforce 0` |
| timezone | Asia/Shanghai | `timedatectl set-timezone Asia/Shanghai` |
| chronyd | delay < 1s when etcd election happen | `vim /etc/chronyc.conf` |
| docker | 19.03.9 or higher | `docker version` |
| kubenetes |  default version 1.20.11 | `kubectl version` |
| CNI | flannel 0.13.0 or higher | `kubectl get pods -n kube-system` |

2. deploy detail

There are many machines in the management platform, and ansible is needed to batch operation machines to save time. It is necessary to deploy root free from the deployment node to other nodes.

```
node_ips={master,node1,node2,...}
test -f /root/.ssh/id_rsa || ssh-keygen -N '' -t rsa -f /root/.ssh/id_rsa  # create ssh key if not exist 
# add ssh key to other nodes, and input password manually
for ip in $node_ips; do
  ssh-copy-id "$ip" || { echo "failed on $ip."; break; }  #exit if failed
done
```
3. deploy
#### 1) offline deploy
```
$ yum -y localinstall ansible-2.9.27-1.el7.ans.noarch.rpm
```

#### 2) check version
```
$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.6/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.6.8 (default, Nov 16 2020, 16:55:22) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
```

#### 3) config ansible
```
$ vim /etc/ansible/hosts

[master]
10.3.174.228    node_name=master

[worker]
10.3.174.234    node_name=node1
10.3.174.237    node_name=node2
10.3.174.251    node_name=node3

[etcd]
10.3.174.228    etcd_name=master

[k8s:children]
master
worker
```
5) disable host key checking
```
$ vi /etc/ansible/ansible.cfg
# modify below config
# uncomment this to disable SSH key host checking
host_key_checking = False
``` 
6) disable SELINUX and firewalld
```
$ ansible k8s -m command -a "setenforce 0"
$ ansible k8s -m command -a "sed --follow-symlinks -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config"
$ ansible k8s -m command -a "firewall-cmd --set-default-zone=trusted"
$ ansible k8s -m command -a "firewall-cmd --complete-reload"
$ ansible k8s -m command -a "swapoff -a"
```
7) hosts config
```
$ cd /export/upload && vim hosts_set.sh
# set script content below

#!/bin/bash
cat > /etc/hosts << EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.3.174.228   master
10.3.174.234   node1
10.3.174.237   node2
10.3.174.251   node3
EOF

$ ansible new_worker -m copy -a 'src=/iflytek/upload/hosts_set.sh dest=/export/upload'
$ ansible new_worker -m command -a 'sh /iflytek/upload/hosts_set.sh'
```
