Jika Anda memiliki **2 Control Plane**, **1 etcd**, dan **3 Worker**, Anda perlu menyesuaikan konfigurasi agar Kubernetes berjalan dalam mode **high availability (HA)**. Berikut langkah-langkahnya:

---

## **1. Topologi Cluster**
- **1 etcd node** → Penyimpanan data Kubernetes
- **2 Control Plane** → Menjalankan komponen utama Kubernetes (API Server, Controller, Scheduler)
- **3 Worker** → Menjalankan workload aplikasi

### **IP dan Hostname (Contoh)**
```
192.168.1.10  control-plane-1
192.168.1.11  control-plane-2
192.168.1.20  etcd
192.168.1.30  worker-1
192.168.1.31  worker-2
192.168.1.32  worker-3
192.168.1.40  vip-k8s
```
- `vip-k8s` adalah **Virtual IP** yang digunakan oleh HAProxy atau Keepalived.

---

## **2. Persiapan di Semua Node**
Lakukan langkah-langkah berikut di **semua node**:

### **2.1. Matikan Swap, Atur Firewall, dan Aktifkan IP Forwarding**
```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

### **2.2. Instalasi Container Runtime (Containerd)**
```bash
sudo dnf install -y containerd.io
```
Konfigurasi Containerd:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```
Restart containerd:
```bash
sudo systemctl restart containerd
sudo systemctl enable --now containerd
```

### **2.3. Instalasi Kubernetes (Kubeadm, Kubelet, Kubectl)**
```bash
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

---

## **3. Konfigurasi etcd (Hanya di Node etcd)**
Jalankan di **Node etcd** (`etcd`):

```bash
sudo dnf install -y etcd
sudo systemctl enable --now etcd
```

Edit konfigurasi `/etc/etcd/etcd.conf`:
```ini
ETCD_NAME="etcd"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.20:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.20:2379"
ETCD_INITIAL_CLUSTER="etcd=http://192.168.1.20:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
```

Restart etcd:
```bash
sudo systemctl restart etcd
```

---

## **4. Konfigurasi HA (HAProxy atau Keepalived)**
Agar kedua Control Plane bisa diakses dengan **satu IP virtual**, gunakan **HAProxy** atau **Keepalived**.

### **4.1. Instalasi HAProxy (Di Semua Control Plane)**
```bash
sudo dnf install -y haproxy
```

Edit `/etc/haproxy/haproxy.cfg`:
```ini
frontend kubernetes
    bind *:6443
    default_backend k8s_control_plane

backend k8s_control_plane
    balance roundrobin
    server control-plane-1 192.168.1.10:6443 check
    server control-plane-2 192.168.1.11:6443 check
```

Restart HAProxy:
```bash
sudo systemctl enable --now haproxy
```

---

## **5. Inisialisasi Control Plane**
Jalankan di **Control Plane pertama** (`control-plane-1`):

```bash
sudo kubeadm init --control-plane-endpoint "vip-k8s:6443" --upload-certs --etcd-servers "http://192.168.1.20:2379" --pod-network-cidr=10.0.0.0/16
```

Simpan perintah **join control plane** untuk digunakan nanti.

Konfigurasi **kubectl**:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## **6. Join Control Plane Kedua**
Jalankan di **Control Plane kedua** (`control-plane-2`):

Gunakan perintah join yang diberikan oleh `kubeadm init`, misalnya:
```bash
sudo kubeadm join vip-k8s:6443 --token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx --control-plane --certificate-key yyyyyyyyyyyyyyyyyyyyyyyyyyyy
```

---

## **7. Instalasi Cilium CNI**
Jalankan di **Control Plane** (`control-plane-1`):

```bash
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.14/install/kubernetes/quick-install.yaml
```

Verifikasi:
```bash
kubectl -n kube-system get pods -l k8s-app=cilium
```

---

## **8. Join Worker ke Cluster**
Jalankan di **masing-masing Worker** (`worker-1`, `worker-2`, `worker-3`):

Gunakan perintah dari `kubeadm init`, contoh:
```bash
sudo kubeadm join vip-k8s:6443 --token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## **9. Verifikasi Cluster**
Jalankan di **Control Plane**:

```bash
kubectl get nodes
kubectl get pods -A
```

---

## **Kesimpulan**
Setelah konfigurasi ini selesai, cluster Kubernetes Anda akan memiliki:
✅ **2 Control Plane** (HA dengan HAProxy)  
✅ **1 etcd Node** (Menyimpan data Kubernetes)  
✅ **3 Worker Node** (Menjalankan workload)  
✅ **Cilium CNI** (Untuk jaringan pod)  

Silakan dicoba, dan beri tahu saya jika ada kendala! 🚀