---
layout: post
---
Install kubernetes cluster with ansible  

### Requirements

---
#### 1. description

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

### Start from master Node 
#### 1. config public key authentication
There are many machines in the management platform, and ansible is needed to batch operation machines to save time. It is necessary to deploy root free from the deployment node to other nodes.
``` shell
# modify node_ips to your nodes, and run below script
node_ips="master_ip node1_ip node2_ip ..."
test -f /root/.ssh/id_rsa || ssh-keygen -N '' -t rsa -f /root/.ssh/id_rsa  # create ssh key if not exist 
# add ssh key to other nodes, and input password manually
for ip in $node_ips; do
  ssh-copy-id "$ip" || { echo "failed on $ip."; break; }  #exit if failed
done
```
#### 2. Install Ansible 
``` shell
yum -y localinstall ansible-2.9.27-1.el7.ans.noarch.rpm
```
``` shell
$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.6/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.6.8 (default, Nov 16 2020, 16:55:22) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
```

#### 3. config ansible
``` shell
# modify hosts file
$ vim /etc/ansible/hosts
[master]
10.3.174.228    node_name=master

[worker]
10.3.174.234    node_name=node1
10.3.174.237    node_name=node2
10.3.174.251    node_name=harbor01

[etcd]
10.3.174.228    etcd_name=master

[k8s:children]
master
worker
```
#### 4. disable host key checking
``` shell
$ vi /etc/ansible/ansible.cfg
# modify below config
# uncomment this to disable SSH key host checking
host_key_checking = False
``` 

#### 5. make directory of work,log,data,upload
``` shell
ansible k8s -m command -a "mkdir -p /iflytek/server"
ansible k8s -m command -a "mkdir -p /iflytek/log"
ansible k8s -m command -a "mkdir -p /iflytek/data"
ansible k8s -m command -a "mkdir -p /iflytek/upload"
ansible k8s -m command -a "ls -l /iflytek"
```

#### 6. kernel and network config
```shell
vi /iflytek/upload/system_set.sh
```
add content below to system_set.sh
```shell
#!/bin/bash
sed -ri.k8s.bak '/k8s config begin/,/k8s config end/d' /etc/sysctl.conf
    cat >> "/etc/sysctl.conf" << EOF
# k8s config begin
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
# k8s config end
EOF
    sysctl --system
    # ulimit
    sed -ri.k8s.bak '/k8s config begin/,/k8s config end/d' /etc/security/limits.conf
    cat >> /etc/security/limits.conf << EOF
# k8s config begin
* soft memlock unlimited
* hard memlock unlimited
* soft nproc 102400
* hard nproc 102400
* soft nofile 1048576
* hard nofile 1048576
# k8s config end 
EOF
```
run script
``` shell
ansible k8s -m copy -a 'src=/iflytek/upload/system_set.sh dest=/iflytek/upload'
ansible k8s -m command -a 'sh /iflytek/upload/system_set.sh'
```

disable SELINUX and firewalld
``` shell
ansible k8s -m command -a "setenforce 0"
ansible k8s -m command -a "sed --follow-symlinks -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config"
ansible k8s -m command -a "firewall-cmd --set-default-zone=trusted"
ansible k8s -m command -a "firewall-cmd --complete-reload"
ansible k8s -m command -a "swapoff -a"
```
add hosts
```shell
vi /iflytek/upload/hosts_set.sh
```
add script content below
``` shell
#!/bin/bash
cat > /etc/hosts << EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.3.174.228   master
10.3.174.234   node1
10.3.174.237   node2
10.3.174.251   harbor01
EOF
```
run script
``` shell
ansible k8s -m copy -a 'src=/iflytek/upload/hosts_set.sh dest=/iflytek/upload'
ansible k8s -m command -a 'sh /iflytek/upload/hosts_set.sh'
```
### Install docker 19.03.9 on master

#### 1. install docker rpm
``` shell
# upload docker-rpm.tar.gz to /iflytek/upload and unzip it
tar -zxvf docker-rpm.tgz -C /iflytek/upload/
# install docker
yum -y localinstall /iflytek/upload/docker-rpm/*
``` 
#### 2. start docker and check version
``` shell
systemctl start docker 
systemctl status docker
systemctl enable docker
```
#### 3. check docker version
``` shell 
$ docker version
Client: Docker Engine - Community
 Version:           19.03.9
 API version:       1.40
 Go version:        go1.13.10
 Git commit:        9d988398e7
 Built:             Fri May 15 00:25:27 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.9
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.10
  Git commit:       9d988398e7
  Built:            Fri May 15 00:24:05 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.9
  GitCommit:        e25210fe30a0a703442421b0f60afac609f950a3
 runc:
  Version:          1.0.1
  GitCommit:        v1.0.1-0-g4144b63
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```
#### 4. install docker-compose v2.9.0
``` shell
# grant execute permission
cp /iflytek/upload/docker-rpm/docker-compose /usr/local/bin/
chmod +x /usr/local/bin/docker-compose
# check docker-compose version
$ docker-compose version
Docker Compose version v2.9.0
```


### Install docker for all nodes

---
#### 1. upload docker-rpm.tgz to /iflytek/upload
``` shell
$ ls /iflytek/upload/docker-rpm.tgz
# distribute package
$ ansible k8s -m copy -a "src=/iflytek/upload/docker-rpm.tgz dest=/iflytek/upload/"
# expected result
CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum": "acd3897edb624cd18a197bcd026e6769797f4f05", 
    "dest": "/iflytek/upload/docker-rpm.tgz", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "3ba6d9fe6b2ac70860b6638b88d3c89d", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "system_u:object_r:usr_t:s0", 
    "size": 103234394, 
    "src": "/root/.ansible/tmp/ansible-tmp-1661836788.82-13591-17885284311930/source", 
    "state": "file", 
    "uid": 0
}
```

#### 2. unzip docker-rpm.tgz and install docker
``` shell
ansible k8s -m shell -a "tar -zxvf /iflytek/upload/docker-rpm.tgz -C /iflytek/upload/ && yum -y localinstall /iflytek/upload/docker-rpm/*"
```
#### 3. enable boot up && start docker
``` shell
ansible k8s -m shell -a "systemctl enable docker && systemctl start docker"
```
#### 4. check docker version
``` shell
ansible k8s -m shell -a "docker version"
# expected result
Client: Docker Engine - Community
 Version:           19.03.9
 API version:       1.40
 Go version:        go1.13.10
 Git commit:        9d988398e7
 Built:             Fri May 15 00:25:27 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.9
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.10
  Git commit:       9d988398e7
  Built:            Fri May 15 00:24:05 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.9
  GitCommit:        e25210fe30a0a703442421b0f60afac609f950a3
 runc:
  Version:          1.0.1
  GitCommit:        v1.0.1-0-g4144b63
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```

### install harbor v2.4.3 on harbor node
#### 1. upload harbor package 
``` shell

# unzip harbor-offline-installer-v2.4.3.tgz
scp harbor-offline-installer-v2.4.3.tgz root@harbor_ip:/iflytek/upload/
ssh root@harbor_ip
cd /iflytek/upload/
tar -zxvf harbor-offline-installer-v2.4.3.tgz -C /iflytek/server/
cd /iflytek/server/harbor/
``` 
#### 2. config harbor.yml as below
``` shell
# modify harbor.cfg
$ cp harbor.yml.tmpl harbor.yml
$ vi harbor.yml
```
``` shell
hostname: harbor_ip
data_volume: /iflytek/data/harbor
log.location: /iflytek/log/harbor
certificate: /iflytek/data/ssl/harbor01.crt
private_key: /iflytek/data/ssl/harbor01.key
```
#### 3. generate ssl certificate
``` shell
mkdir -p /iflytek/data/ssl
cd /iflytek/data/ssl
# generate ca.key && ca.pem
$ openssl genrsa -out ca.key 3072
$ openssl req -new -x509 -days 3650 -key ca.key -out ca.pem

Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Hefei
Locality Name (eg, city) [Default City]:Hefei
# other info can be empty

# generate harbor01.key && harbor01.csr for domain name
$ openssl genrsa -out harbor01.key 3072
$ openssl req -new -key harbor01.key -out harbor01.csr
```
show as below
``` shell
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Hefei
Locality Name (eg, city) [Default City]:Hefei
Common Name (eg, your name or your server's hostname) []:hostname  
```
sign harbor01.csr with ca.key && ca.pem
``` shell
$ openssl x509 -req -in harbor01.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out harbor01.pem -days 3650
```
show as below
``` shell
Signature ok
subject=C = CN, ST = Hefei, L = Hefei, CN = hostname
Getting CA Private Key
```

#### 4. import harbor images
``` shell
$ docker load -i harbor.v2.4.3.tar.gz 
# wait to import images
$ docker images
REPOSITORY                      TAG       IMAGE ID       CREATED       SIZE
goharbor/harbor-exporter        v2.4.3    776ac6ee91f4   4 weeks ago   81.5MB
goharbor/chartmuseum-photon     v2.4.3    f39a9694988d   4 weeks ago   172MB
goharbor/redis-photon           v2.4.3    b168e9750dc8   4 weeks ago   154MB
goharbor/trivy-adapter-photon   v2.4.3    a406a715461c   4 weeks ago   251MB
goharbor/notary-server-photon   v2.4.3    da89404c7cf9   4 weeks ago   109MB
goharbor/notary-signer-photon   v2.4.3    38468ac13836   4 weeks ago   107MB
goharbor/harbor-registryctl     v2.4.3    61243a84642b   4 weeks ago   135MB
goharbor/registry-photon        v2.4.3    9855479dd6fa   4 weeks ago   77.9MB
goharbor/nginx-photon           v2.4.3    0165c71ef734   4 weeks ago   44.4MB
goharbor/harbor-log             v2.4.3    57ceb170dac4   4 weeks ago   161MB
goharbor/harbor-jobservice      v2.4.3    7fea87c4b884   4 weeks ago   219MB
goharbor/harbor-core            v2.4.3    d864774a3b8f   4 weeks ago   197MB
goharbor/harbor-portal          v2.4.3    85f00db66862   4 weeks ago   53.4MB
goharbor/harbor-db              v2.4.3    7693d44a2ad6   4 weeks ago   225MB
goharbor/prepare                v2.4.3    c882d74725ee   4 weeks ago   268MB
```

#### 5. start harbor
``` shell
./prepare # if harbor.yml changed，execute it update
./install.sh --help
./install.sh --with-chartmuseum
```

---
### Install kubernetes 

```shell
$ mkdir -p /iflytek/repo/kubeadm-rpm
$ yum install -y kubelet-1.20.11 kubeadm-1.20.11 kubectl-1.20.11 --downloadonly --downloaddir=/iflytek/repo/kubeadm-rpm
```

#### 1. upload kubeadm-rpm to k8s master node
```shell
$ ls /iflytek/upload/
kubeadm-rpm.tar.gz
$ ansible k8s -m copy -a "src=/iflytek/upload/kubeadm-rpm.tgz dest=/iflytek/upload/kubeadm-rpm.tgz"
```
show as below
```shell
CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum": "3fe96fe1aa7f4a09d86722f79f36fb8fde69facb", 
    "dest": "/export/upload/kubeadm-rpm.tgz", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "80d5bda420db6ea23ad75dcf0f76e858", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "system_u:object_r:usr_t:s0", 
    "size": 67423355, 
    "src": "/root/.ansible/tmp/ansible-tmp-1661840257.4-33361-139823848282879/source", 
    "state": "file", 
    "uid": 0
}
```
#### 2. install kubeadm-rpm
```shell
$ ansible k8s -m shell -a "tar -zxvf /iflytek/upload/kubeadm-rpm.tgz -C /iflytek/upload/"
$ ansible k8s -m shell -a "yum install -y /iflytek/upload/kubeadm-rpm/*.rpm"
```
#### 3. set boot up
```shell
$ ansible k8s -m shell -a "systemctl enable kubelet && systemctl start kubelet"
$ journalctl -xefu kubelet
```

#### 4. distribute dependency images
```shell

docker push docker.kxdigit.com/library/coredns:v1.8.4
docker push docker.kxdigit.com/library/etcd:3.5.0-0
docker push docker.kxdigit.com/library/kube-apiserver:v1.22.4
docker push docker.kxdigit.com/library/kube-controller-manager:v1.22.4
docker push docker.kxdigit.com/library/kube-proxy:v1.22.4
docker push docker.kxdigit.com/library/kube-scheduler:v1.22.4
docker push docker.kxdigit.com/library/pause:3.5
docker push docker.kxdigit.com/library/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
docker push docker.kxdigit.com/library/mirrored-flannelcni-flannel:v0.19.1
```
```shell
kubeadm init \
--control-plane-endpoint "10.132.10.91:6443" \
--image-repository 10.132.10.100/community \
--kubernetes-version v1.22.4 \
--service-cidr=172.16.0.0/16 \
--pod-network-cidr=10.244.0.0/16 \
--token "abcdef.0123456789abcdef" \
--token-ttl "0" \
--upload-certs
```
