# Kubernetes v1.25 installation notes

---

## init ENV

---

vagrant up 1 master node & 2 worker nodes
check hostname with:

~~~bash
  hostnamectl
~~~

### VMs

~~~text
  ubuntu2004master01 - (Mac: 080027277FF6)
  ubuntu2004master02 - (Mac: )
  ubuntu2004worker01 - (Mac: 080027B4E7C8)
  ubuntu2004worker02 - (Mac: 080027AE457A)
~~~

### ssh key

~~~bash
  sudo ssh-copy-id vagrant@10.0.0.100
  sudo ssh-copy-id vagrant@10.0.0.200
  sudo ssh-copy-id vagrant@10.0.0.201
  sudo ssh-copy-id vagrant@10.0.0.202
~~~

### add /etc/hosts

~~~text
  10.0.0.100 ubuntu2004master01
  10.0.0.200 ubuntu2004master02
  10.0.0.201 ubuntu2004worker01
  10.0.0.202 ubuntu2004worker02
~~~

### disable swap

check with:

~~~bash
  free
  swapon -s
~~~

disable with:

~~~bash
  swapoff -a
  sed -i '/swap/s/^/#/' /etc/fstab
~~~

or

~~~bash
  sudo systemctl disable --now swap.img.swap
  sudo systemctl mask swap.target
~~~

### time sync (optinal)

~~~bash
  apt -y install chrony
  chrony sources -v
~~~

### disable firewall (optinal)

~~~bash
  ufw disable
  ufw status
~~~

### docker uses systemd

check docker Cgroup driver: systemd

~~~bash
  docker info | grep Cgroup
~~~

~~~bash
  sudo vi /etc/docker/daemon.json
~~~

~~~json
  {
  "exec-opts": ["native.cgroupdriver=systemd"]
  }
~~~

~~~bash
  sudo systemctl restart docker.service
~~~

---

## install kubeadm kubelet kubectl

---

Reference Doc

- <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux>
- <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm>

1. Update the apt package index and install packages needed to use the Kubernetes apt repository:

    ~~~bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    ~~~

2. Download the Google Cloud public signing key:

    ~~~bash
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    ~~~

3. Add the Kubernetes apt repository:

    ~~~bash
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ~~~

4. Update apt package index with the new repository:

    ~~~bash
    sudo apt-get update
    ~~~

5. Check available version:

    ~~~bash
    apt-cache madison kubeadm|head
    apt-cache madison kubelet|head
    apt-cache madison kubectl|head
    ~~~

6. Install kubeadm kubelet kubectl:

    ~~~bash
    sudo apt-get install -y kubeadm kubelet kubectl
    ~~~

7. Hold kubeadm kubelet kubectl version to block automatic updates

    ~~~bash
    sudo apt-mark hold kubelet kubeadm kubectl
    ~~~

---

## install cri-dockerd

---

Project: <https://github.com/Mirantis/cri-dockerd>

~~~bash
  sudo dpkg -i cri-dockerd_0.2.5.3-0.ubuntu-focal_amd64.deb
~~~

---

## init Kubernetes cluster with kubeadm init

---

Reference Doc

- <https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm>

### Master Node

~~~bash
  sudo kubeadm init \
  --apiserver-advertise-address 10.0.0.100 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///run/cri-dockerd.sock\
  --token-ttl=0 \
  --upload-certs
~~~

Response

>Your Kubernetes control-plane has initialized successfully!
>To start using your cluster, you need to run the following as a regular user:

~~~bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

>Alternatively, if you are the root user, you can run:

~~~bash
  export KUBECONFIG=/etc/kubernetes/admin.conf
~~~

>You should now deploy a pod network to the cluster.
>Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
><https://kubernetes.io/docs/concepts/cluster-administration/addons/>

### install CNI - Calico

Reference Doc

- <https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart>

1. Install the Tigera Calico operator and custom resource definitions.

    ~~~bash
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml
    ~~~

2. Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest, see the installation reference.

    ~~~bash
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/custom-resources.yaml
    ~~~

3. Confirm that all of the pods are running with the following command.

    ~~~bash
    kubectl get pods -n calico-system
    ~~~

    Wait until each pod has the ***STATUS*** of ***Running.***

### Worker Node

Response from ***kubeadm init***

>Then you can join any number of worker nodes by running the following on each as root:

~~~bash
sudo kubeadm join 10.0.0.100:6443 --token 70q2ho.b62bdamrzj4nye1j --discovery-token-ca-cert-hash sha256:05f4ce3c999c970ef31a4c2ade45b1ce0aa215c2ec6e5898af72fb1bd3b896dd --cri-socket unix:///run/cri-dockerd.sock
~~~

---

## kubectl autocompletion

---

Reference Doc

- <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion>

~~~bash
  sudo apt-get install bash-completion
  source /usr/share/bash-completion/bash_completion
  echo 'source <(kubectl completion bash)' >>~/.bashrc
  exec bash
~~~

---

## Reset cluster

---

***worker*** node first, then ***master*** node

~~~bash
  sudo kubeadm reset -f --cri-socket unix:///run/cri-dockerd.sock
  rm -rf /etc/cni/net.d/ $HOME/.kube/config
  reboot
~~~

---

## BVT

---

~~~bash
  kubectl create deployment demoapp --image=ikubernetes/demoapp:v1.0 --replicas=2
  kubectl create service nodeport demoapp --tcp=80:80
  kubectl get svc
  curl ${demoapp ip}
~~~

Response

~~~bash
  iKubernetes demoapp v1.0 !! ClientIP: 10.0.0.100, ServerName: demoapp-55c5f88dcb-p5rf9, ServerIP: 192.168.210.130!
  iKubernetes demoapp v1.0 !! ClientIP: 10.0.0.100, ServerName: demoapp-55c5f88dcb-p64l8, ServerIP: 192.168.243.194!
~~~
