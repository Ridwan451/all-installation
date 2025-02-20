Untuk menghapus **kubelet**, **kubectl**, **kubeadm**, dan **containerd** di **Rocky Linux 8.10**, ikuti langkah-langkah berikut:  

---

### **1️⃣ Hentikan Layanan Kubelet & Containerd**  
```bash
systemctl stop kubelet
systemctl stop containerd
```

---

### **2️⃣ Hapus Paket Kubernetes (kubelet, kubectl, kubeadm)**
```bash
dnf remove -y kubelet kubectl kubeadm
```

---

### **3️⃣ Hapus Containerd**
```bash
dnf remove -y containerd
```

---

### **4️⃣ Hapus Konfigurasi Kubernetes & Data Node**
```bash
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /var/lib/cni /var/lib/containerd
rm -rf ~/.kube
```

---

### **5️⃣ Hapus Modul Kernel & Reset Konfigurasi IPTables**
```bash
modprobe -r br_netfilter
iptables -F && iptables -X
iptables -t nat -F && iptables -t nat -X
iptables -t raw -F && iptables -t raw -X
iptables -t mangle -F && iptables -t mangle -X
```

---

### **6️⃣ Bersihkan Repositori Kubernetes**
Jika sebelumnya menginstal Kubernetes dari repo, hapus repository-nya:
```bash
rm -f /etc/yum.repos.d/kubernetes.repo
dnf clean all
```

---

### **7️⃣ Reboot Server**
```bash
reboot
```

Setelah reboot, worker-1 akan benar-benar bersih dari Kubernetes dan Containerd.  
Silakan coba langkah ini dan beri tahu jika ada kendala! 🚀