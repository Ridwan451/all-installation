**install kubernetes di rocky 8.10**
---

Jalankan perintah berikut di semua node (master dan worker):
```bash
sudo dnf update -y
sudo dnf install -y vim git curl wget bash-completion
```

---

Matikan Swap
```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

sudo setenforce 0
sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

Do this on the workers only.
```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp  
sudo firewall-cmd --reload
```

In addition, we’ll have to open the firewall ports for Cilium.
Do this on all nodes.
```bash
sudo firewall-cmd --permanent --add-port=4240/tcp
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --reload
```

Aktifkan Modul Kernel
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Atur parameter sysctl:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

tambahkan repo containerd dan Install containerd
```bash
sudo dnf config-manager --set-enabled powertools
sudo dnf config-manager --set-enabled appstream
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io --allowerasing
```

Konfigurasi containerd:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Edit file /etc/containerd/config.toml, ubah SystemdCgroup menjadi true:
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart containerd:
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Install Kubernetes (kubeadm, kubelet, kubectl)**
Tambahkan repository Kubernetes:
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
```

Install paket Kubernetes:
```bash
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

cek version
```bash
kubectl version
```
