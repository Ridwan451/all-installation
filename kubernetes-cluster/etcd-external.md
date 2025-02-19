**install etcd-external di rocky 8.10**
---

Jalankan perintah berikut di semua node (master dan worker):
```bash
sudo dnf update -y
sudo dnf install -y vim git curl wget bash-completion
```

---

### **Install etcd di VM-ETCD**
#### **A. Unduh dan Install etcd**
Jalankan perintah berikut di **vm-etcd** untuk mengunduh dan menginstal etcd versi terbaru yang kompatibel:  
```bash
ETCD_VERSION="v3.5.10"
ARCH="amd64"

# Download binary etcd
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-${ARCH}.tar.gz

# Ekstrak file
tar -xvf etcd-${ETCD_VERSION}-linux-${ARCH}.tar.gz

# Pindahkan binary ke /usr/local/bin
sudo mv etcd-${ETCD_VERSION}-linux-${ARCH}/etcd* /usr/local/bin/

# Verifikasi instalasi
etcd --version
```
Pastikan outputnya menunjukkan versi **v3.5.10** atau versi yang diinstal.

---

#### **B. Buat User & Direktori etcd**
```bash
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd

sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```

---

### **Konfigurasi etcd**
#### **A. Tentukan IP VM-ETCD**
Jalankan:
```bash
ip a
```
Misalnya, IP `vm-etcd` adalah **`172.25.214.5`**.

---

#### **B. Buat File Konfigurasi etcd**
Buat file konfigurasi untuk **systemd**:
```bash
sudo tee /etc/systemd/system/etcd.service > /dev/null <<EOF
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
Type=notify
User=etcd
Group=etcd
ExecStart=/usr/local/bin/etcd \\
  --name vm-etcd \\
  --data-dir /var/lib/etcd \\
  --initial-advertise-peer-urls http://172.25.214.5:2380 \\
  --listen-peer-urls http://172.25.214.5:2380 \\
  --advertise-client-urls http://172.25.214.5:2379 \\
  --listen-client-urls http://172.25.214.5:2379,http://127.0.0.1:2379 \\
  --initial-cluster vm-etcd=http://172.25.214.5:2380 \\
  --initial-cluster-token etcd-cluster-1 \\
  --initial-cluster-state new
Restart=always
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF
```
**Ganti `172.25.214.5` dengan IP vm-etcd Anda.**  

---

### **Jalankan dan Aktifkan etcd**
Jalankan service etcd:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
```
Verifikasi statusnya:
```bash
sudo systemctl status etcd
```
Pastikan statusnya **active (running)**.

Cek apakah etcd berjalan dengan benar:
```bash
ETCDCTL_API=3 etcdctl --endpoints=http://172.25.214.5:2379 endpoint health
```
Output yang benar:
```
{"health":"true"}
```

---

### **Konfigurasi Kubernetes agar Menggunakan etcd External**
Di setiap **control-plane (control-plane-1 dan control-plane-2)**, edit file **`/root/kubeadm-config.yaml`** agar menggunakan **vm-etcd**:
```yaml
etcd:
  external:
    endpoints:
      - http://172.25.214.5:2379
```
Lalu jalankan:
```bash
kubeadm init --config=/root/kubeadm-config.yaml
```

---

### **Kesimpulan**
1. **Install etcd** di **vm-etcd**.
2. **Konfigurasi service** dengan IP **vm-etcd**.
3. **Jalankan service** dan cek kesehatan etcd.
4. **Ubah konfigurasi Kubernetes** agar menggunakan **etcd external**.

Jika ada kendala, kirimkan hasil dari:
```bash
sudo systemctl status etcd
journalctl -u etcd --no-pager | tail -20
```

---

Ya, **etcd external membutuhkan sertifikat TLS** jika Anda ingin Kubernetes **control-plane** (master) terhubung ke **etcd external** dengan **aman**.  

Namun, **jika Anda menggunakan koneksi HTTP tanpa enkripsi**, maka **sertifikat tidak diperlukan**, tetapi ini **tidak disarankan** untuk lingkungan produksi karena **tidak aman**.

---

## **1. Kapan Sertifikat Diperlukan?**
- **Jika etcd menggunakan TLS (port default: `2379` dan `2380`)**  
- **Jika Kubernetes control-plane perlu koneksi aman ke etcd**  
- **Jika Anda menjalankan cluster multi-node dengan komunikasi terenkripsi**  

## **2. Kapan Sertifikat Tidak Diperlukan?**
- **Jika Anda menggunakan HTTP tanpa TLS** (`http://<etcd-ip>:2379`)  
- **Jika hanya untuk pengujian atau lingkungan development**  

---

## **3. Konfigurasi etcd dengan TLS (Disarankan untuk Produksi)**  
Jika Anda ingin menggunakan TLS, Anda perlu membuat sertifikat **CA, client, dan server**.

### **A. Buat Sertifikat TLS untuk etcd**
Di **vm-etcd**, jalankan:
```bash
mkdir -p /etc/etcd/ssl
cd /etc/etcd/ssl
```

#### **1) Buat CA (Certificate Authority)**
```bash
openssl genrsa -out ca-key.pem 2048
openssl req -x509 -new -nodes -key ca-key.pem -subj "/CN=etcd-ca" -days 365 -out ca.pem
```

#### **2) Buat Sertifikat Server etcd**
```bash
openssl genrsa -out etcd-server-key.pem 2048
openssl req -new -key etcd-server-key.pem -subj "/CN=etcd-server" -out etcd-server.csr
openssl x509 -req -in etcd-server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out etcd-server.pem -days 365
```

#### **3) Buat Sertifikat Client untuk Kubernetes Control-Plane**
```bash
openssl genrsa -out etcd-client-key.pem 2048
openssl req -new -key etcd-client-key.pem -subj "/CN=etcd-client" -out etcd-client.csr
openssl x509 -req -in etcd-client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out etcd-client.pem -days 365
```

---

### **B. Konfigurasi etcd untuk Menggunakan TLS**
Edit **`/etc/systemd/system/etcd.service`** dan ubah konfigurasi:
```ini
ExecStart=/usr/local/bin/etcd \
  --name vm-etcd \
  --data-dir /var/lib/etcd \
  --cert-file=/etc/etcd/ssl/etcd-server.pem \
  --key-file=/etc/etcd/ssl/etcd-server-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd-server.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-server-key.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --listen-peer-urls https://172.25.214.5:2380 \
  --listen-client-urls https://172.25.214.5:2379 \
  --advertise-client-urls https://172.25.214.5:2379 \
  --initial-advertise-peer-urls https://172.25.214.5:2380 \
  --initial-cluster vm-etcd=https://172.25.214.5:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster-state new
```
Kemudian, restart etcd:
```bash
sudo systemctl daemon-reload
sudo systemctl restart etcd
```

---

### **C. Konfigurasi Kubernetes untuk Menggunakan etcd External dengan TLS**
Di **control-plane**, edit **`/root/kubeadm-config.yaml`**:
```yaml
etcd:
  external:
    endpoints:
      - https://172.25.214.5:2379
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd/etcd-client.pem
    keyFile: /etc/kubernetes/pki/etcd/etcd-client-key.pem
```
Pastikan Anda menyalin sertifikat **`ca.pem`**, **`etcd-client.pem`**, dan **`etcd-client-key.pem`** ke **control-plane** di path **`/etc/kubernetes/pki/etcd/`**.

Lalu, jalankan:
```bash
kubeadm init --config=/root/kubeadm-config.yaml
```

---

## **Kesimpulan**
- **Tanpa TLS** → Bisa menggunakan HTTP, **tidak butuh sertifikat**, tapi tidak aman.  
- **Dengan TLS** → **Disarankan untuk produksi**, memerlukan sertifikat agar komunikasi aman.  
- **Konfigurasi Kubernetes** → Harus disesuaikan dengan penggunaan TLS atau HTTP.  

Jika ingin keamanan tinggi, **gunakan TLS** untuk menghindari serangan MITM (Man-in-the-Middle). 🚀