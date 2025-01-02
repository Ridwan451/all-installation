Untuk mengaktifkan **Cilium CNI** untuk **containerd**, Anda harus memastikan bahwa **containerd** dikonfigurasi untuk mendukung plugin CNI, lalu menginstal dan mengatur **Cilium** sebagai plugin jaringan untuk Kubernetes. Berikut adalah panduan langkah demi langkah:

---

### **Langkah 1: Persiapan Sistem**
#### a. **Matikan Plugin CNI Lama**
Jika Anda telah menggunakan plugin jaringan lain (misalnya Flannel atau Calico), hapus konfigurasinya terlebih dahulu untuk mencegah konflik.

```bash
kubectl delete -f <plugin_network_config.yaml>  # Hapus konfigurasi jaringan lama
```

#### b. **Pastikan Kernel Mendukung eBPF**
Cilium menggunakan eBPF, jadi kernel Anda harus mendukungnya. Pastikan kernel Linux Anda adalah versi **4.19** atau lebih tinggi.
```bash
uname -r  # Periksa versi kernel
```

Jika kernel mendukung eBPF, lanjutkan. Jika tidak, Anda perlu memperbarui kernel Anda.

#### c. **Perbarui containerd**
Pastikan Anda telah menginstal **containerd** versi terbaru.
```bash
sudo apt update
sudo apt install -y containerd
```

---

### **Langkah 2: Instalasi Cilium**
#### a. **Pasang CLI Cilium**
Pasang CLI Cilium di node master Anda.
```bash
curl -L --remote-name https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64
chmod +x cilium-linux-amd64
sudo mv cilium-linux-amd64 /usr/local/bin/cilium
```

#### b. **Validasi Cluster**
Periksa apakah cluster Kubernetes Anda siap untuk menggunakan Cilium:
```bash
cilium status
```

Jika ada masalah, ikuti langkah-langkah pemecahan masalah yang diberikan oleh perintah.

---

### **Langkah 3: Konfigurasi Containerd untuk CNI**
#### a. **Tambahkan Konfigurasi Containerd**
Periksa konfigurasi containerd di `/etc/containerd/config.toml`. Jika file ini belum ada, buat satu dengan:
```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit bagian **plugins."io.containerd.grpc.v1.cri".cni** untuk menunjuk ke direktori konfigurasi CNI yang digunakan Cilium (biasanya `/etc/cni/net.d`):
```toml
[plugins."io.containerd.grpc.v1.cri".cni]
  bin_dir = "/opt/cni/bin"        # Lokasi biner plugin CNI
  conf_dir = "/etc/cni/net.d"     # Lokasi file konfigurasi CNI
```

Restart containerd untuk menerapkan perubahan:
```bash
sudo systemctl restart containerd
```

---

### **Langkah 4: Deploy Cilium ke Cluster**
#### a. **Install Cilium**
Gunakan CLI Cilium untuk menginstal Cilium di cluster:
```bash
cilium install
```

#### b. **Verifikasi Instalasi**
Periksa apakah Cilium telah berhasil diterapkan:
```bash
cilium status
kubectl get pods -n kube-system
```

Anda harus melihat pod Cilium yang berjalan di namespace `kube-system`.

---

### **Langkah 5: Konfigurasi Jaringan dengan Cilium**
#### a. **Enable eBPF**
Aktifkan eBPF di Cilium untuk performa terbaik:
```bash
cilium status --wait
cilium config set enable-ebpf true
```

#### b. **Konfigurasi Network Policy**
Opsional: Terapkan kebijakan jaringan dengan Cilium jika Anda memerlukan kontrol jaringan granular.

Contoh:
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-http
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: backend
    toPorts:
    - ports:
      - port: 80
        protocol: TCP
```

---

### **Langkah 6: Verifikasi Fungsionalitas**
#### a. **Cek Status Pod**
Periksa apakah semua pod dapat berkomunikasi sesuai konfigurasi jaringan:
```bash
kubectl get pods -A
kubectl get nodes
```

#### b. **Observabilitas dengan Hubble**
Cilium menyediakan observabilitas menggunakan **Hubble**. Instal Hubble:
```bash
cilium hubble enable
cilium hubble port-forward&
```

Gunakan CLI Hubble untuk memantau lalu lintas jaringan:
```bash
hubble status
hubble observe
```

---

### **Catatan Penting**
1. **Kernel Modern:** Cilium memanfaatkan eBPF, jadi kernel modern sangat penting.
2. **CNI Konfigurasi:** Pastikan tidak ada plugin CNI lain yang aktif saat menggunakan Cilium.
3. **Observabilitas Opsional:** Hubble memungkinkan Anda memantau jaringan secara real-time.

Jika ada pertanyaan tambahan, beri tahu saya! 😊