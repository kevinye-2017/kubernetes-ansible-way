# kubernetes-ansible-way
本文以ansible来完高可用k8s集群,以kubeadm方式来完成集群的初始化。kubeadm对多网卡支持不好，etcd、k8s相关组件会使用默认出口网关ip网段作为集群ip。所以正式环境还是建议使用二进制部署，或者对kubeadm部署扣成二进制!

### 环境准备

- kubernetes v1.15.4 (haproxy、keepalived)
- etcd v3.3.10
- flannel v0.11.0-amd64
- docker-ce (建议18.09.9,19.03+ kubelet会提示其它问题)
- ansible 2.8+ (因为用到docker_image模块，需要pip install docker-py)
- mitogen-0.2.8 (ansible 提升执行速度模块，理论上能达到2-7倍)

ansible可能需要额外的依赖库，如

```
pip install ansible==2.8.5
pip install docker-py
yum install sshpass git -y
```

mitogen的详情可以参照[mitogen](https://networkgenomics.com/ansible/) 

```
mkdir -pv /etc/ansible
wget https://networkgenomics.com/try/mitogen-0.2.8.tar.gz --no-check-certificate -O - |tar xz  -C /etc/ansible/

cat >> /etc/ansible/ansible.cfg <<EOF
[defaults]
strategy_plugins = /etc/ansible/mitogen-0.2.8/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
host_key_checking = False
EOF
```



### 运行ansible roles

1.git clone 到本地

```
git clone https://github.com/kevinye-2017/kubernetes-ansible-way.git
cd kubernetes-ansible-way
```

2.配置ansible
这里使用的虚拟ip会在一些云环境没法达到自动漂移，可以将LoadBlanceIP 换成SLB。

- 这里的ssh默认端口均为22，如果为其它端口需修改ansible inventory (hosts) ansible_ssh_port=port
- 主机名会设定hosts里的名称
- roles里的变量全部定义在kubernetes-installation/defaults/main.yaml，还有一些变量会在运行时进行动态register，这部分在kubernetes-installation/tasks/init-master.yaml

3.运行ansible roles，master集群
这一部分会先设定好HA高可用、3台master自动创建并加入集群

```
ansible-playbook -i hosts kubernetes-installation.yaml
```

![cluster-info](https://github.com/kevinye-2017/Images-use-for-readme/blob/master/k8s/cluster-info.PNG)

4.后续添加ndoe节点

```
ansible-playbook -i hosts add-node.yaml
```
5.添加lables

```
kubectl label node node1 node2 node3 node-role.kubernetes.io/worker=
```



### 一些errors/issue 
https://github.com/kontena/pharos-cluster/issues/440
