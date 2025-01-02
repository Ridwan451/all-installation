Menginstal **Kubernetes Vanilla** berarti Anda akan menggunakan versi Kubernetes resmi tanpa modifikasi dari vendor pihak ketiga. Berikut langkah-langkah untuk menginstalnya secara manual:

---

### **1. Persiapan**
#### a. **Persyaratan Sistem**
- **Sistem Operasi:** Linux (disarankan Ubuntu/Debian/CentOS).
- **CPU:** Minimal 2 vCPU.
- **RAM:** Minimal 2 GB (untuk node worker) dan 4 GB (untuk control plane).
- **Penyimpanan:** Minimal 20 GB.

#### b. **Komponen Utama**
1. **kubeadm:** Alat untuk bootstrap cluster Kubernetes.
2. **kubelet:** Agen yang berjalan di setiap node untuk menjalankan pod.
3. **kubectl:** CLI untuk mengelola Kubernetes.

#### c. **Jaringan**
- Pastikan semua node dapat saling berkomunikasi (port 6443, 2379, 2380, 10250, 10251, dan 10252 terbuka).
- Pilih plugin jaringan **CNI** seperti Flannel, Calico, atau Cilium.

---

### **2. Instalasi Kubernetes Vanilla**
#### a. **Update dan Instalasi Prasyarat**
Pada semua node (control plane dan worker):
```bash
sudo apt update
sudo apt install -y apt-transport-https curl
```

#### b. **Tambahkan Repository Kubernetes**
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt update
```

#### c. **Instal kubeadm, kubelet, dan kubectl**
```bash
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### **3. Konfigurasi Cluster Kubernetes**
#### a. **Matikan Swap**
Swap harus dinonaktifkan untuk menjalankan Kubernetes.
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

#### b. **Aktifkan Modul Kernel yang Dibutuhkan**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

---

### **4. Bootstrap Cluster dengan kubeadm**
#### a. **Inisialisasi Control Plane**
Pada node master:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

#### b. **Konfigurasi kubectl**
Setelah inisialisasi berhasil:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### c. **Pasang Plugin Jaringan**
Pasang CNI plugin (contoh: Flannel):
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

---

### **5. Tambahkan Worker Node**
Pada setiap node worker:
1. Jalankan perintah dari output `kubeadm init` di master node, seperti:
   ```bash
   kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```
2. Jika token habis masa berlakunya, buat token baru di node master:
   ```bash
   kubeadm token create --print-join-command
   ```

---

### **6. Verifikasi Cluster**
1. Cek status node:
   ```bash
   kubectl get nodes
   ```
2. Cek pod yang berjalan:
   ```bash
   kubectl get pods -A
   ```

---

### **7. (Opsional) Instal Dashboard Kubernetes**
Untuk akses visual:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

---

### **Catatan Penting**
1. **Pilih Plugin Jaringan:** Sesuaikan plugin jaringan dengan kebutuhan Anda (Flannel, Calico, atau lainnya).
2. **High Availability (Opsional):** Untuk produksi, tambahkan lebih banyak control plane dan gunakan load balancer.
3. **Backup Konfigurasi:** Pastikan Anda menyimpan konfigurasi `admin.conf` dengan aman untuk pengelolaan cluster.
