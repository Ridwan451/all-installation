### 🔍 **Penyebab Masalah**  
Jika setelah reboot port **10250 masih digunakan**, kemungkinan besar **kubelet masih otomatis berjalan** saat sistem menyala kembali.

---

## **🛠 1️⃣ Cek Proses yang Menggunakan Port 10250 Lagi**
Pastikan proses mana yang masih menggunakan port **10250** setelah reboot:

```bash
netstat -tulnp | grep 10250
```
Atau:
```bash
ss -tulnp | grep 10250
```
Jika output menunjukkan ada proses yang menggunakan port **10250**, cari tahu **PID (Process ID)** dan matikan.

**Contoh output:**
```
tcp   LISTEN   0   4096   0.0.0.0:10250   0.0.0.0:*   users:(("kubelet",pid=12345,fd=21))
```
Jika proses tersebut adalah `kubelet`, matikan paksa:
```bash
kill -9 12345  # Ganti dengan PID yang muncul di output
```

---

## **🛠 2️⃣ Hentikan Kubelet dari Systemd**  
Karena `kubelet` otomatis dijalankan oleh **Systemd**, hentikan dan nonaktifkan agar tidak berjalan saat reboot:

```bash
systemctl stop kubelet
systemctl disable kubelet
systemctl mask kubelet
```
Kemudian pastikan kubelet **tidak berjalan kembali setelah reboot**:

```bash
systemctl list-units --type=service | grep kubelet
```
Jika masih muncul, coba hapus dari systemd:
```bash
rm -f /etc/systemd/system/kubelet.service
rm -rf /etc/systemd/system/kubelet.service.d
systemctl daemon-reload
```

Lalu pastikan status kubelet:
```bash
systemctl status kubelet
```
Pastikan **tidak ada lagi yang berjalan**.

---

## **🛠 3️⃣ Cek Apakah Ada Container Kubelet yang Masih Berjalan**
Jika masih ada proses di **port 10250**, bisa jadi ada container yang masih berjalan. Coba cek:

```bash
crictl ps -a | grep kubelet
```
Atau jika menggunakan `docker`:
```bash
docker ps -a | grep kubelet
```
Jika ada, hapus container tersebut:
```bash
crictl rm $(crictl ps -aq)  # Jika menggunakan containerd
docker rm -f $(docker ps -aq)  # Jika menggunakan Docker
```
Lalu pastikan tidak ada lagi yang berjalan:
```bash
crictl ps -a
```

---

## **🛠 4️⃣ Bersihkan Semua Konfigurasi Kubernetes**
Jika masih bermasalah, hapus semua konfigurasi Kubernetes agar bersih total:

```bash
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /var/lib/cni /etc/cni/net.d ~/.kube
```

Lalu **restart sistem**:
```bash
reboot
```

---

## **🛠 5️⃣ Jalankan Kembali `kubeadm init`**
Setelah reboot, pastikan kubelet **tidak berjalan** dengan:
```bash
systemctl status kubelet
```
Jika sudah tidak ada yang menggunakan **port 10250**, jalankan kembali:
```bash
kubeadm init --config=kubeadm-config.yaml
```

---

### **🚀 Kesimpulan**  
✔ **Hentikan kubelet dengan `systemctl stop kubelet` dan nonaktifkan agar tidak otomatis berjalan (`disable` & `mask`)**  
✔ **Hapus semua proses yang masih berjalan dengan `kill -9 PID`**  
✔ **Cek dan hapus container yang masih berjalan (`crictl ps -a` atau `docker ps -a`)**  
✔ **Pastikan setelah reboot, kubelet tidak berjalan kembali**  
✔ **Bersihkan sisa konfigurasi Kubernetes dan coba init ulang**

Coba langkah-langkah di atas, lalu beritahu saya jika masih ada masalah! 🚀