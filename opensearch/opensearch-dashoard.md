Bagus, OpenSearch sudah berjalan! 🎉 Sekarang, kita akan menginstal **OpenSearch Dashboards** agar bisa mengakses OpenSearch melalui antarmuka web.  

---

## **1. Download dan Install OpenSearch Dashboards**  
Jalankan perintah berikut di VM **Rocky Linux 8.10**:  

```bash
sudo yum install -y https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.13.0/opensearch-dashboards-2.13.0-linux-x64.rpm
```
> **Pastikan versinya sesuai dengan OpenSearch yang sudah diinstall!** Gantilah `2.13.0` dengan versi OpenSearch yang sedang kamu gunakan.

---

## **2. Konfigurasi OpenSearch Dashboards**  
Edit file konfigurasi:  
```bash
sudo nano /etc/opensearch-dashboards/opensearch_dashboards.yml
```
Ubah atau tambahkan parameter berikut:  
```yaml
server.host: "0.0.0.0"
opensearch.hosts: ["http://<IP-VM>:9200"]
opensearch.ssl.verificationMode: none
opensearch.username: "kibanaserver"
opensearch.password: "kibanaserver"
opensearch.requestHeadersWhitelist: ["authorization", "securitytenant"]
opensearch_security.enabled: false
opensearch_security.multitenancy.enabled: false
opensearch_security.multitenancy.tenants.preferred: ["Private", "Global"]
opensearch_security.readonly_mode.roles: []
opensearch_security.cookie.secure: false
```  

Simpan dan keluar (`CTRL + X`, lalu `Y`, lalu `Enter`).

---  

### 3. **Buat Direktori yang Hilang**  
Jalankan perintah berikut untuk membuat direktori yang diperlukan:  
```bash
mkdir -p /usr/share/opensearch-dashboards/config/
```

### 4. **Buat Symbolic Link Lagi**  
Setelah direktori dibuat, jalankan ulang perintah:  
```bash
ln -s /etc/opensearch-dashboards/opensearch_dashboards.yml /usr/share/opensearch-dashboards/config/opensearch_dashboards.yml
```

### 5. **Pastikan Izin dan Kepemilikan File**  
Pastikan OpenSearch Dashboards dapat membaca file konfigurasinya:  
```bash
chown -R opensearch-dashboards:opensearch-dashboards /usr/share/opensearch-dashboards/config/
chmod 750 /usr/share/opensearch-dashboards/config/
chmod 640 /etc/opensearch-dashboards/opensearch_dashboards.yml
```

### 6. **Restart OpenSearch Dashboards**  
Coba restart OpenSearch Dashboards:  
```bash
systemctl restart opensearch-dashboards
```
Lalu cek statusnya:  
```bash
systemctl status opensearch-dashboards -l
```

---

## **7. Jalankan OpenSearch Dashboards**
Aktifkan dan mulai service:  
```bash
sudo systemctl enable --now opensearch-dashboards
```
Cek statusnya:  
```bash
sudo systemctl status opensearch-dashboards
```

---

## **8. Akses OpenSearch Dashboards**  
Buka browser dan akses:  
```
http://<IP_VM>:5601
```
**Contoh:**  
Jika IP VM **172.31.212.23**, akses di browser:  
```
http://172.31.212.23:5601
```

---

### **Jika Tidak Bisa Diakses**
1. **Pastikan Firewall Mengizinkan Port 5601**  
   ```bash
   sudo firewall-cmd --permanent --add-port=5601/tcp
   sudo firewall-cmd --reload
   ```

2. **Cek Log Jika Ada Error**  
   ```bash
   journalctl -u opensearch-dashboards --no-pager | tail -n 50
   ```

---

**menonaktifkan autentikasi** di OpenSearch Dashboards dengan memastikan **OpenSearch Security Plugin benar-benar tidak aktif**.  

---

## **1. Pastikan OpenSearch Security Dinonaktifkan di OpenSearch**
Edit file **`opensearch.yml`**:  
```bash
sudo nano /etc/opensearch/opensearch.yml
```
Pastikan baris berikut ada:  
```yaml
plugins.security.disabled: true
```
Simpan (`Ctrl + X`, lalu `Y`, lalu `Enter`).

---

## **2. Nonaktifkan Security Plugin di OpenSearch Dashboards**  
Edit file **`opensearch_dashboards.yml`**:  
```bash
sudo nano /etc/opensearch-dashboards/opensearch_dashboards.yml
```
Tambahkan atau ubah baris berikut:  
```yaml
opensearch_security.enabled: false
opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: []
```
Simpan (`Ctrl + X`, lalu `Y`, lalu `Enter`).

---

## **3. Hapus Plugin Security Dashboards**
Karena Anda tidak menemukan plugin `opensearch-security`, coba hapus **`securityDashboards`** sepenuhnya:  
```bash
sudo rm -rf /usr/share/opensearch-dashboards/plugins/securityDashboards
```
Atau bisa juga dengan cara ini:  
```bash
sudo mv /usr/share/opensearch-dashboards/plugins/securityDashboards /usr/share/opensearch-dashboards/plugins/securityDashboards.bak
```
Langkah ini mencegah OpenSearch Dashboards memuat plugin keamanan.

---

## **4. Hapus Cache Plugin OpenSearch Dashboards**
```bash
sudo rm -rf /usr/share/opensearch-dashboards/optimize/bundles
```

---

## **5. Restart OpenSearch dan OpenSearch Dashboards**
```bash
sudo systemctl restart opensearch
sudo systemctl restart opensearch-dashboards
```

---

## **6. Cek Status Layanan**
Pastikan semua berjalan normal:  
```bash
sudo systemctl status opensearch
sudo systemctl status opensearch-dashboards
```

---

## **7. Akses OpenSearch Dashboards**
Coba buka **http://10.10.11.2:5601** di browser.  
Jika masih meminta login, coba akses **tanpa "https"**:  
```
http://10.10.11.2:5601
```
dan gunakan username `admin` serta password default `admin`.

---

Setelah ini, kamu bisa login ke OpenSearch Dashboards dan mulai eksplorasi! 🚀  
Kalau ada masalah, kasih tahu error-nya, biar kita selesaikan. 😃