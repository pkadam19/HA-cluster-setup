# HA-cluster-setup
HA cluster setup using HAProxy

### What is HAproxy:
High Availability Proxy (HAProxy) is high performance and open source TCP/HTTP load balancer. It is mostly used to improve the performance and reliability of applications by distributing the workload across the underlying server. 
### Role of HAPorxy here:
We are using HAProxy to balance the load between the API master endpoints.


### Pre-requisities

| Role |	FQDN |	IP |	OS | RAM |	CPU |
| ---- | ----- | ---- | ----- | ----- | ----- |
| LoadBalancer | loadbalancer.example.com | 192.168.16.200 | centos 7 | 2 GB | 1 CPU |
| Master | master1.example.com | 192.168.16.101 | centos 7 | 4 GB | 2 CPU |
| Master | master2.example.com | 192.168.16.102 | centos 7 | 4 GB | 2 CPU |
| Master | master3.example.com | 192.168.16.103 | centos 7 | 4 GB | 2 CPU |
| Worker | worker1.example.com | 192.168.16.111 | centos 7 | 2 GB | 1 CPU |
| Worker | worker2.example.com | 192.168.16.112 | centos 7 | 2 GB | 1 CPU |

Update `/etc/hosts` file with below
  ```
  192.168.16.200	loadbalancer.example.com	loadbalancer
  192.168.16.101	master1.example.com	master1
  192.168.16.102	master2.example.com	master2
  192.168.16.103	master3.example.com	master3
```

### Setup LoadBalancer

Perform all commands on loadbalancer nodes

* Login to the loadbalancer node

* Switch as root - `sudo su -`

* Install HAProxy
  ```yum install haproxy -y```

* Configure HAProxy
  ```
  global
   log /dev/log  local0
   log /dev/log  local1 notice
   stats socket /var/lib/haproxy/stats level admin
   chroot /var/lib/haproxy
   user haproxy
   group haproxy
   daemon

  defaults
   log global
   mode  http
   option  httplog
   option  dontlognull
         timeout connect 5000
         timeout client 50000
         timeout server 50000

  frontend kubernetes
     bind 192.168.16.200:6443
     option tcplog
     mode tcp
     default_backend kubernetes-master-nodes

  backend kubernetes-master-nodes
     mode tcp
     balance roundrobin
     option tcp-check

     server master1 192.168.16.101:6443 check
     server master2 192.168.16.102:6443 check
   server master3 192.168.16.103:6443 check
  ```

* Start HAProxy Service

  ```
  systemctl start haproxy
  systemctl enable haproxy
  ```

* Test the connection
Test it on local systemctl

  ```
  [root@loadbalancer ~]# telnet loadbalancer 6443

  Trying 192.168.16.200...
  Connected to loadbalancer.
  Escape character is '^]'.

  ```
  OR
  ```
  [root@loadbalancer ~]# nc -v loadbalancer 6443

  Ncat: Version 7.50 ( https://nmap.org/ncat )
  Ncat: Connected to 192.168.16.200:6443.

  ```

  **NOTE:** If you see failures for master1, master2 and master3 connectivity, you can ignore them for time being as we have not yet installed anything on the servers.



### Perform on all master and worker nodes

Perform all the commands as root user unless otherwise specified

* Disable Firewall
  ```
  systemctl disable firewalld; systemctl stop firewalld
  ```
* Disable swap
  ```
  swapoff -a; sed -i '/swap/d' /etc/fstab
  ```
* Disable SELinux
  ```
  setenforce 0
  sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
  ```
* Update sysctl settings for Kubernetes networking
  ```
  cat >>/etc/sysctl.d/kubernetes.conf<<EOF
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  sysctl --system
  ```
* Install docker engine
  ```
  yum install -y yum-utils device-mapper-persistent-data lvm2
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  yum install -y docker-ce-19.03.12
  systemctl enable --now docker
  ```
#### Kubernetes Setup
* Add yum repository
  ```
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
  ```
* Install Kubernetes components
  ```
  yum install -y kubeadm-1.18.5-0 kubelet-1.18.5-0 kubectl-1.18.5-0
  ```
* Enable and Start kubelet service
  ```
  systemctl enable --now kubelet
  ```

  **NOTE:**  Ensure all nodes have same version of docker, kubeadm, kubectl, kubelet.

### Configure kubeadm to bootstrap the cluster 

We will start off by initializing only one master node. For this demo, we will use master1 to initialize our first control plane.

##### Log in to master1
  * Switch to root account - sudo -i
  * Execute the below command to initialize the cluster -

    ```
    kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs --pod-network-cidr=192.168.0.0/16
    ```
    It becomes
    ```
    kubeadm init --control-plane-endpoint "loadbalancer:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
    ```
    Output looks like
    ```
    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.

    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    You can now join any number of the control-plane node running the following command on each as root:

    kubeadm join 192.168.16.200:6443 --token gwor71.7al53bcle93cb3p5 \
      --discovery-token-ca-cert-hash sha256:f5c9979518b640489849aeeb44714b8b133f4de65b7cc15389c6b944842a63ce \
      --control-plane --certificate-key e600e916597a15c6a0a7cdfadee4015ba6b408711de7f1fea5f356d244ed576c

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
    "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 192.168.16.200:6443 --token gwor71.7al53bcle93cb3p5 \
      --discovery-token-ca-cert-hash sha256:f5c9979518b640489849aeeb44714b8b133f4de65b7cc15389c6b944842a63ce
    ```

    The output consists of 3 major tasks -

    1. Setup kubeconfig using -
      ```
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
      ```

    2. Setup new control plane (master) using
      ```
      kubeadm join 192.168.16.200:6443 --token gwor71.7al53bcle93cb3p5 \
        --discovery-token-ca-cert-hash sha256:f5c9979518b640489849aeeb44714b8b133f4de65b7cc15389c6b944842a63ce \
        --control-plane --certificate-key e600e916597a15c6a0a7cdfadee4015ba6b408711de7f1fea5f356d244ed576c
      ```

    3. Join worker node using
      ```
      kubeadm join 192.168.16.200:6443 --token gwor71.7al53bcle93cb3p5 \
        --discovery-token-ca-cert-hash sha256:f5c9979518b640489849aeeb44714b8b133f4de65b7cc15389c6b944842a63ce
      ```

      **NOTE**

       Your output will be different than what is provided here. While performing the rest of the demo, ensure that you are executing the command provided by your output and dont copy and paste from here.

       Save the output in some secure file for future use.

  * Setup kubeconfig using below command and verify the kubectl
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    Verify kubectl
    ```
    kubectl get nodes
    NAME      STATUS      ROLES    AGE     VERSION
    master1   NotReady    master   9h      v1.18.5
    ```
    It is fine, status is in NotReady state, as we havent configured the CNI yet.

  ##### Log in to master2
  * Switch to root account - sudo -i
  * Execute the below command to initialize the cluster -
    ```
    kubeadm join 192.168.16.200:6443 --token gwor71.7al53bcle93cb3p5 \
      --discovery-token-ca-cert-hash sha256:f5c9979518b640489849aeeb44714b8b133f4de65b7cc15389c6b944842a63ce \
      --control-plane --certificate-key e600e916597a15c6a0a7cdfadee4015ba6b408711de7f1fea5f356d244ed576c
    ```
    Output looks like
    ```
    [mark-control-plane] Marking the node master3 as control-plane by adding the label "node-role.kubernetes.io/master=''"
    [mark-control-plane] Marking the node master3 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

     This node has joined the cluster and a new control plane instance was created:

     * Certificate signing request was sent to apiserver and approval was received.
     * The Kubelet was informed of the new secure connection details.
     * Control plane (master) label and taint were applied to the new node.
     * The Kubernetes control plane instances scaled up.
     * A new etcd member was added to the local/stacked etcd cluster.
    ```
  * Setup kubeconfig using below command and verify the kubectl
      ```
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      ```
      Verify cluster
      ```
      kubectl get nodes
      NAME      STATUS      ROLES    AGE     VERSION
      master1   NotReady    master   21m     v1.18.5
      master2   NotReady    master   15m     v1.18.5
      ```
  ##### Log in to master3
   * Switch to root account - sudo -i
   * Execute the below command to initialize the cluster -
     ```
      kubeadm join 192.168.16.200:6443 --token gwor71.7al53bcle93cb3p5 \
        --discovery-token-ca-cert-hash sha256:f5c9979518b640489849aeeb44714b8b133f4de65b7cc15389c6b944842a63ce \
        --control-plane --certificate-key e600e916597a15c6a0a7cdfadee4015ba6b408711de7f1fea5f356d244ed576c
     ```
     Output looks like
     ```
     [mark-control-plane] Marking the node master3 as control-plane by adding the label "node-role.kubernetes.io/master=''"
     [mark-control-plane] Marking the node master3 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

      This node has joined the cluster and a new control plane instance was created:

      * Certificate signing request was sent to apiserver and approval was received.
      * The Kubelet was informed of the new secure connection details.
      * Control plane (master) label and taint were applied to the new node.
      * The Kubernetes control plane instances scaled up.
      * A new etcd member was added to the local/stacked etcd cluster.
     ```
   * Setup kubeconfig using below command and verify the kubectl
      ```
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      ```
      Verify kubectl
      ```
      kubectl get nodes
      NAME      STATUS      ROLES    AGE     VERSION
      master1   NotReady    master   21m     v1.18.5
      master2   NotReady    master   15m     v1.18.5
      master3   NotReady    master   9m17s   v1.18.5
      ```

  ##### Log in to Worker1 and Worker2
   * Switch to root account - sudo -i
   * Execute the below command to initialize the cluster -
     ```
      kubeadm join 192.168.16.200:6443 --token gwor71.7al53bcle93cb3p5 \
        --discovery-token-ca-cert-hash sha256:f5c9979518b640489849aeeb44714b8b133f4de65b7cc15389c6b944842a63ce
     ```
     Output looks like
     ```
      This node has joined the cluster:
       * Certificate signing request was sent to apiserver and a response was received.
       * The Kubelet was informed of the new secure connection details.

       Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
     ```

### Configure kubeconfig on loadbalancer node

* Log in to loadbalancer node
* Switch to root - sudo -i
* Create a directory - .kube at $HOME of root
  ```
  mkdir -p $HOME/.kube
  ```
* SCP configuration file from any one master node to loadbalancer node
  ```
  scp master1:/etc/kubernetes/admin.conf $HOME/.kube/config
  ```
* Provide appropriate ownership to the copied file
  ```
  chown $(id -u):$(id -g) $HOME/.kube/config
  ```
* Install kubectl binary
  ```
  yum install kubectl-1.18.5-0 -y
  ```
* Verify the cluster
  ```
  kubectl get nodes
  NAME      STATUS      ROLES    AGE     VERSION
  master1   NotReady    master   21m     v1.18.5
  master2   NotReady    master   15m     v1.18.5
  master3   NotReady    master   9m17s   v1.18.5
  worker1   NotReady    <none>   3m17s   v1.18.5
  worker1   NotReady    <none>   2m55s   v1.18.5
  ```

### Install CNI and complete installation

We are executing the below command from LoadBalancer node
 ```
 export kubever=$(kubectl version | base64 | tr -d '\n')
 kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
 ```
 This will install CNI components all over nodes.

 Verfify the Cluster
 ```
  kubectl get nodes
  NAME      STATUS   ROLES    AGE     VERSION
  master1   Ready    master   9h      v1.18.5
  master2   Ready    master   9h      v1.18.5
  master3   Ready    master   9h      v1.18.5
  worker1   Ready    <none>   9h      v1.18.5
  worker2   Ready    <none>   4h25m   v1.18.5
  ```
Verfiy CS
  ```
  kubectl get cs

  NAME                 STATUS    MESSAGE             ERROR
  controller-manager   Healthy   ok
  scheduler            Healthy   ok
  etcd-0               Healthy   {"health":"true"}
  ```
Verify Pods
  ```
  kubectl get pods -n kube-system

  NAME                              READY   STATUS    RESTARTS   AGE
  coredns-66bff467f8-f26nd          1/1     Running   0          5h47m
  etcd-master1                      1/1     Running   0          9h
  etcd-master2                      1/1     Running   0          9h
  etcd-master3                      1/1     Running   0          9h
  kube-apiserver-master1            1/1     Running   1          9h
  kube-apiserver-master2            1/1     Running   0          9h
  kube-apiserver-master3            1/1     Running   0          9h
  kube-controller-manager-master1   1/1     Running   5          9h
  kube-controller-manager-master2   1/1     Running   4          9h
  kube-controller-manager-master3   1/1     Running   2          9h
  kube-proxy-jq9c8                  1/1     Running   0          4h26m
  kube-proxy-llg52                  1/1     Running   0          9h
  kube-proxy-ppmhv                  1/1     Running   0          9h
  kube-proxy-slhdq                  1/1     Running   0          9h
  kube-proxy-xdq6c                  1/1     Running   0          9h
  kube-scheduler-master1            1/1     Running   5          9h
  kube-scheduler-master2            1/1     Running   4          9h
  kube-scheduler-master3            1/1     Running   2          9h
  weave-net-bxphl                   2/2     Running   0          4h26m
  weave-net-mbcsl                   2/2     Running   1          5h55m
  weave-net-pcjln                   2/2     Running   1          5h59m
  weave-net-pvskh                   2/2     Running   4          6h7m
  weave-net-qvrsl                   2/2     Running   0          5h39m
  ```

Your HA cluster is ready!!

