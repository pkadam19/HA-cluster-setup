## Single Master K8s setup (1 Master & 2 worker nodes)

This documentation is for centos 7 where we are setting up a cluster with one master node and two worker node.

### Pre-requisities

| Role |	FQDN |	IP |	OS | RAM |	CPU |
| ---- | ----- | ---- | ----- | ----- | ----- |
| Master | master1.example.com | 172.16.16.100 | centos 7 | 4 GB | 2 CPU |
| Worker | worker1.example.com | 172.16.16.103 | centos 7 | 2 GB | 1 CPU |
| Worker | worker2.example.com | 172.16.16.104   | centos 7 | 2 GB | 1 CPU |

If you are using vagrant the refer the sample Vagrantfile

### Perform on master and worker nodes

You can perform all commands with root user. 

* Disable Firewall
  ```
  sudo systemctl disable firewalld; sudo systemctl stop firewalld
  ```
* Disable swap
  ```
  sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab
  ```
* Disable SELinux
  ```
  sudo setenforce 0
  sudo sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
  ```
* Update sysctl settings for Kubernetes networking
  ```
  sudo tee -a /etc/sysctl.d/kubernetes.conf << 'EOF'
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  sudo sysctl --system
  ```
* Install docker engine
  ```
  sudo yum install -y yum-utils device-mapper-persistent-data lvm2
  sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  sudo yum install -y docker-ce-19.03.12 
  sudo systemctl enable --now docker
  ```
#### Kubernetes Setup
* Add yum repository
  ```
  sudo tee -a /etc/yum.repos.d/kubernetes.repo << 'EOF'
  [kubernetes]
  name=Kubernetes
  baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
          https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  EOF
  ```
* Install Kubernetes components
  ```
  sudo yum install -y kubeadm-1.18.5-0 kubelet-1.18.5-0 kubectl-1.18.5-0
  ```
* Enable and Start kubelet service
  ```
  systemctl enable --now kubelet
  ```

### Perform on master node

We are initializing the k8s cluster here.

* To initialize cluster:
  apiserver-advertise-address: Master IP itself
  Use sudo here as kubeadm demands PrivilegedUser
  ```
  sudo kubeadm init --apiserver-advertise-address=172.16.16.100 --pod-network-cidr=192.168.0.0/16
  ```
  Output looks like:
  ```
  Your Kubernetes control-plane has initialized successfully!

  To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

  You should now deploy a pod network to the cluster.
  Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

  Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 172.16.16.100:6443 --token ljvfea.sq2cxijjw1uik7lz \
      --discovery-token-ca-cert-hash sha256:3d0daafee2a5b00682bd265eb28d91c77c67f66774b855580c59964fd50e9580
  ```
  This Output conatins mainly two things:
  * Worker node join command
    ```
    kubeadm join 172.16.16.100:6443 --token ljvfea.sq2cxijjw1uik7lz \
        --discovery-token-ca-cert-hash sha256:3d0daafee2a5b00682bd265eb28d91c77c67f66774b855580c59964fd50e9580
    ```
  * To run `kubectl` commands with regular user.
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
  * Kubelet Status:
  
    You can check the kubelet status: 
    ```
    sudo systemctl status kubelet.service
    ```
    Service should be up and running.

### Perform on worker node
We are joining worker nodes to the cluster.

* Use the output from 'kubeadm init' command from master node.
  ```
  sudo kubeadm join 172.16.16.100:6443 --token ljvfea.sq2cxijjw1uik7lz \
      --discovery-token-ca-cert-hash sha256:3d0daafee2a5b00682bd265eb28d91c77c67f66774b855580c59964fd50e9580
  ```
  Output looks like:
  ```
  This node has joined the cluster:
  * Certificate signing request was sent to apiserver and a response was received.
  * The Kubelet was informed of the new secure connection details.

  Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
  ```

OR
  
* Create new join token using below command (Execute on MASTER node)
  ```
  sudo kubeadm token create --print-join-command
  ```
  Execute the output of this command on worker node.

### Perform on master node:

##### To run kubectle command without sudo
  ```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
##### Verfiy the cluster 
* Get Nodes status
  ```
  [vagrant@master1 ~]$ kubectl get nodes
  NAME                  STATUS     ROLES    AGE     VERSION
  master1.example.com   NotReady   master   21m     v1.18.5
  worker1.example.com   NotReady   <none>   16m     v1.18.5
  worker2.example.com   NotReady   <none>   2m22s   v1.18.5
  ```
* Get cluster components
  ```
  [vagrant@master1 ~]$ kubectl get cs
  NAME                 STATUS    MESSAGE             ERROR
  controller-manager   Healthy   ok                  
  scheduler            Healthy   ok                  
  etcd-0               Healthy   {"health":"true"}
  ```

##### Deploy network on cluster
* Before applying the CNI, 
  Expected behavior:
  ```
  [vagrant@master1 ~]$ kubectl get pods -n kube-system
  NAME                                          READY   STATUS    RESTARTS   AGE
  coredns-66bff467f8-h5gjz                      0/1     Pending   0          26m
  coredns-66bff467f8-wwtp5                      0/1     Pending   0          26m
  etcd-master1.example.com                      1/1     Running   0          26m
  kube-apiserver-master1.example.com            1/1     Running   0          26m
  kube-controller-manager-master1.example.com   1/1     Running   0          26m
  kube-proxy-8hmt4                              1/1     Running   0          26m
  kube-proxy-jd86p                              1/1     Running   0          7m25s
  kube-proxy-k6t2n                              1/1     Running   0          21m
  kube-scheduler-master1.example.com            1/1     Running   0          26m
  ```
* Deploy Network: 
  ```
  kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
  ```

* After deploying the network:
  ```
  [vagrant@master1 ~]$ kubectl get pods -n kube-system
  NAME                                          READY   STATUS    RESTARTS   AGE
  calico-kube-controllers-65f8bc95db-bkw86      1/1     Running   0          14m
  calico-node-2bx4p                             1/1     Running   0          14m
  calico-node-498g7                             1/1     Running   0          14m
  calico-node-wspsb                             1/1     Running   0          14m
  coredns-66bff467f8-h5gjz                      1/1     Running   0          44m
  coredns-66bff467f8-wwtp5                      1/1     Running   0          44m
  etcd-master1.example.com                      1/1     Running   0          44m
  kube-apiserver-master1.example.com            1/1     Running   0          44m
  kube-controller-manager-master1.example.com   1/1     Running   0          44m
  kube-proxy-8hmt4                              1/1     Running   0          44m
  kube-proxy-jd86p                              1/1     Running   0          25m
  kube-proxy-k6t2n                              1/1     Running   0          39m
  kube-scheduler-master1.example.com            1/1     Running   0          44m
  ```
  
 ## Enjoy!!
