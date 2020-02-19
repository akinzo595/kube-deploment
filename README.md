# kube-deploment
                                                 Deploying kubernetes cluster with kubeadm on Centos7.7

Requirements/steps:
Virtualbox 6.0 in my case
Enable two network cards ——> I like to segment network communication between NICs that are internal facing/external facing for easy troubleshooting 

ROLES		  FQDN   OS/Version	RAM	CPU	
master-node kaster.example.com Centos7.7 	2GB  	2GB ( for the master node the RAM/CPU is very important as this costs me time on my initial build)

worker-node		kclient.example.com		Centos7.7  1GB RAM 	iGGB CPU

On both master node and woker node 

Update the /etc/hosts file


cat  >>/etc/hosts<<EOF
192.168.54.4		kmaster.example.com 	kmaster
192.168.54.5		client.example.com		 kclient
EOF

Install docker using docker using docker community edition.

On master:

[dele@kmaster ~]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.57.4	kmaster.example.com	kmaster
192.168.57.5	kclient.example.com	kclient
[dele@kmaster ~]$


On worker:
[dele@kclient .kube]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.57.4	kmaster.example.com	kmaster
192.168.57.5	kclient.example.com	kclient
[dele@kclient .kube]$


yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce 

systemctl enable docker
systemctl start docker 

Disable SElinux

setenforce 0
sed -i —follow-symlink `s/^SELINUX=enforcing/SELINUX=disabled/` /etc/sysconfig/selinux

Disable Firewall

systemctl disable firewalld
systemctl stop firewalld

Install and configure kubernetes

Disable swap
sed -i `swap/d` /etc/fstab
swap off -a

make changes to network settings for kubernetes

cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl —system


Add kubernetes to the repository 

cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

Install Kubernetes 

Yum install -y kubeadm kubelet kubectl

systemctl enable kubelet
systemctl start kubelet

On master node Run the below command

kubeadm init --apiserver-advertise-address=192.168.57.4 --pod-network-cidr=192.168.57.0/24 ( Note 192.168.57.4 is my master node ip address

The above commands will take couple of minutes and if run successfully it should generate a the command to join your worker node to you master 


change user to the user that will be running/interacting with the  kubernetes cluster in this is my user “dele”

[root@kmaster cent7-kubernetes]# su - dele
[dele@kmaster ~]$ mkdir /home/dele/.kube


I opt for flannel network as it appears easy to use and also its one of the ones that I was able to configure pretty easy in my experience.

root@kmaster dele]# wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

As I discussed earlier that I like my network segmented which means I have two important network configured for this setup 

[root@kmaster dele]# ip a | grep -i enp
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic enp0s3
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.57.4/24 brd 192.168.57.255 scope global noprefixroute dynamic enp0s8
[root@kmaster dele]#

Since enpos8 is master node ip . I will have to make a change to the kube-flannel.yml file.

vi kube-flannel.yml ans sers for “flanneld”

I made changes to the below file by adding the highlighted green parameters

containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=enp0s8



[dele@kmaster ~]$ kubectl apply -f kube-flannel.yml


At this time I decided to do  REBOOT of the master server.

Login to the master node and run the below command

[dele@kmaster ~]$ scp /home/dele/.kube/config dele@kclient:/home/dele/.kube/  (the config file is needed on the worker node otherwise you may not be able to deploy pods as needed)

Run this on the worker node:

Kubeadm token create —print-join-command ( this commands will return out put of the command to join a worker node to the cluster in my case it returned the below output. It normally take few minutes before it finally complete.

kubeadm join 192.168.57.4:6443 --token 6ldkfd.kjan6blmz0gv6c9l     --discovery-token-ca-cert-hash sha256:0ea09c9a3c5713a6bdf4d34f402d78d7846c4c2312f27ead30f901ec4e4adf6f


Verifying the cluster 

[dele@kclient .kube]$ !76
kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
[dele@kclient .kube]$ kubectl get nodes
NAME                  STATUS   ROLES    AGE   VERSION
kclient.example.com   Ready    <none>   12h   v1.17.2
kmaster.example.com   Ready    master   20h   v1.17.2
[dele@kclient .kube]$

[dele@kclient .kube]$ kubectl cluster-info
Kubernetes master is running at https://192.168.57.4:6443
KubeDNS is running at https://192.168.57.4:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[dele@kclient .kube]$
 

Thank you! Please let me know what you think. Thanks 

Akindele



