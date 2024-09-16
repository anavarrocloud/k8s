# Architecture
Test Environment

Rocky Linux 9.4, K8s v1.29.9, Containerd, Calico, Istio, Helm, Nfs, Database and Jenkins operator

3 - Master Node: 3x Hp DL20, 64 GB RAM, 128 GB SSD
1 - Worker Node: 1x Dell PowerEdge T430, 190GB RAM, 1.92TB SSD. 

Transformers
Master1: optimus
Master2: ultramagnus
Master3: sentinel
Worker:  bumblebee

# Setting up Kubernetes Cluster

On all K8s Nodes:

dnf update -y
reboot

vim /etc/fstab
sudo swapoff -a
sudo mount -a 

setenforce 0; sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

vi /etc/hosts <-- Place IP Addresses and Hostnames of all Nodes.

modprobe overlay; modprobe br_netfilter; modprobe nf_nat; modprobe xt_REDIRECT; modprobe xt_owner; modprobe iptable_nat; modprobe iptable_mangle; modprobe iptable_filter

# Set Modules

cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

cat > /etc/modules-load.d/istio.conf << EOF
br_netfilter
nf_nat
xt_REDIRECT
xt_owner
iptable_nat
iptable_mangle
iptable_filter
EOF

cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

dnf makecache
dnf install -y containerd.io
mv /etc/containerd/config.toml /etc/containerd/config.toml.orig
containerd config default > /etc/containerd/config.toml
vi /etc/containerd/config.toml
---
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
---
https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md

VERSION="v1.29.0" # check latest version in /releases page
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz

---
sudo vim /etc/crictl.yaml
---

systemctl enable --now containerd.service

firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252,179}/tcp
firewall-cmd --reload

cat > /etc/yum.repos.d/k8s.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

dnf makecache; dnf install -y {kubelet,kubeadm,kubectl} --disableexcludes=kubernetes

systemctl enable --now kubelet.service

source <(kubectl completion bash)
kubectl completion bash > /etc/bash_completion.d/kubectl

---

vi kubeadm-config.yaml

apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.29.9
controlPlaneEndpoint: "optimus:6443"
networking:
  podSubnet: 10.30.0.0/16


kubeadm init --config kubadm-config.yaml

---
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

vi custom-resources.yaml

---
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: IP.of.Choice/Range
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
---

curl -L https://github.com/projectcalico/calico/releases/download/v3.27.0/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
sudo mv calicoctl /usr/bin

---

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

---
curl -L https://istio.io/downloadIstio | sh -
chmod +x istioctl && sudo mv istioctl /usr/bin
istioctl install
kubectl label namespace default istio-injection=enabled

firewall-cmd --permanent --add-port={15000-15009,15010,15012,15014,15017,15020,15021,15053,15090}/tcp
firewall-cmd --reload

sudo kubeadm init --cri-socket unix:///var/run/containerd/containerd.sock --pod-network-cidr 10.30.0.0/16 --control-plane-endpoint qa-master1:6443

Labels:

kubectl label node qa-master1 node-role.kubernetes.io/master=master
kubectl label node qa-worker1 node-role.kubernetes.io/worker=worker
kubectl label node qa-worker2 node-role.kubernetes.io/worker=worker

https://aquasecurity.github.io/kube-bench/v0.6.15/installation/
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.6.15/kube-bench_0.6.15_linux_amd64.rpm -o kube-bench_0.6.15_linux_amd64.rpm
sudo yum install kube-bench_0.6.15_linux_amd64.rpm -y

---
mongodb-operator:
https://github.com/mongodb/mongodb-kubernetes-operator/blob/master/docs/install-upgrade.md
		
---
Jenkins Operator:
https://jenkinsci.github.io/kubernetes-operator/docs/getting-started/latest/installing-the-operator/
---

sudo kubeadm token create --print-join-command
kubectl label node <Hostname> node-role.kubernetes.io/worker=worker

---

# Point at latest release
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
# Deploy the KubeVirt operator
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
# Create the KubeVirt CR (instance deployment request) which triggers the actual installation
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml
# Install Virtctl
export VERSION=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
wget https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
chmod +x virtctl-${VERSION}-linux-amd64
mv virtctl-${VERSION}-linux-amd64 /usr/local/bin/virtctl

