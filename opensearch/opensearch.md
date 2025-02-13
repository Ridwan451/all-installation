Berikut adalah langkah-langkah lengkap untuk menginstal dan menjalankan **OpenSearch** di **Rocky Linux 8.10** dari awal:  

---

## **1. Persiapan Awal**
Sebelum menginstal OpenSearch, pastikan **Rocky Linux 8.10** sudah diperbarui:  
```bash
sudo dnf update -y
```
Kemudian, instal beberapa dependensi yang diperlukan:  
```bash
sudo dnf install -y curl wget unzip tar nano java-17-openjdk
```
Pastikan Java terinstal dengan benar:  
```bash
java -version
```
Output yang diharapkan:  
```
openjdk version "17.0.x" 202x-xx-xx
```

---

## **2. Download dan Install OpenSearch**
Unduh OpenSearch versi terbaru (misalnya versi **2.13.0**):  
```bash
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.13.0/opensearch-2.13.0-linux-x64.tar.gz
```
Ekstrak file yang telah diunduh:  
```bash
tar -xvf opensearch-2.13.0-linux-x64.tar.gz
```
Pindahkan direktori OpenSearch ke lokasi yang lebih sesuai:  
```bash
sudo mv opensearch-2.13.0 /usr/share/opensearch
```
Buat direktori konfigurasi:  
```bash
sudo mkdir -p /etc/opensearch
sudo cp -r /usr/share/opensearch/config/* /etc/opensearch/
```

---

## **3. Buat User dan Group untuk OpenSearch**
Demi keamanan, jalankan OpenSearch dengan user non-root:  
```bash
sudo useradd -r -M -d /usr/share/opensearch -s /sbin/nologin opensearch
```
Ubah kepemilikan folder OpenSearch agar dapat diakses oleh user tersebut:  
```bash
sudo chown -R opensearch:opensearch /usr/share/opensearch
sudo chown -R opensearch:opensearch /etc/opensearch
```

---

## **4. Konfigurasi OpenSearch**
Edit file konfigurasi utama:  
```bash
sudo nano /etc/opensearch/opensearch.yml
```
Tambahkan atau ubah konfigurasi berikut:
```yaml
cluster.name: opensearch-cluster

node.name: node-1

# Jika hanya menggunakan 1 node
discovery.type: single-node

network.host: 0.0.0.0

http.port: 9200

# Port transport (internal cluster communication)
transport.port: 9300

# Keamanan (jika ingin menonaktifkan security plugin, tambahkan ini)
plugins.security.disabled: true
```
Simpan lalu keluar (`CTRL+X`, lalu `Y`, lalu `ENTER`).

---

## **5. Konfigurasi OpenSearch sebagai Service systemd**
Buat file service systemd:  
```bash
sudo nano /etc/systemd/system/opensearch.service
```
Tambahkan konfigurasi berikut:  
```ini
[Unit]
Description=OpenSearch
Documentation=https://opensearch.org/docs/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=opensearch
Group=opensearch
Environment=OPENSEARCH_HOME=/usr/share/opensearch
Environment=OPENSEARCH_PATH_CONF=/etc/opensearch
ExecStart=/bin/bash -c '/usr/share/opensearch/bin/opensearch'
Restart=always
LimitNOFILE=65536
LimitNPROC=4096
LimitAS=infinity
LimitRSS=infinity
WorkingDirectory=/usr/share/opensearch

[Install]
WantedBy=multi-user.target
```
Simpan lalu keluar (`CTRL+X`, lalu `Y`, lalu `ENTER`).

Reload systemd agar mengenali service baru:  
```bash
sudo systemctl daemon-reload
```
Aktifkan service agar berjalan otomatis saat boot:  
```bash
sudo systemctl enable opensearch
```
Jalankan OpenSearch sebagai service:  
```bash
sudo systemctl start opensearch
```
Cek statusnya:  
```bash
sudo systemctl status opensearch
```

---

## **6. Verifikasi Instalasi**
Pastikan OpenSearch berjalan dengan mengakses **REST API**:  
```bash
curl -X GET "http://localhost:9200"
```
Jika berjalan dengan baik, akan muncul output seperti ini:  
```json
{
  "name" : "node-1",
  "cluster_name" : "opensearch-cluster",
  "cluster_uuid" : "xxxxxxxx",
  "version" : {
    "distribution" : "opensearch",
    "number" : "2.13.0",
    "build_type" : "tar",
    ...
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}
```

---

## **7. (Opsional) Buka Port di Firewall**
Jika ingin mengakses OpenSearch dari luar server:  
```bash
sudo firewall-cmd --permanent --add-port=9200/tcp
sudo firewall-cmd --reload
```

---

### **Selesai! 🎉**
Sekarang OpenSearch telah terinstal dan berjalan sebagai **service systemd** di Rocky Linux 8.10.