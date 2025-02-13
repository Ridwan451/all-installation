### **Persyaratan Instalasi OpenSearch di Rocky Linux 8.10**
Sebelum menginstal OpenSearch, pastikan sistem memenuhi persyaratan berikut:

#### **1. Persyaratan Sistem**
- **RAM**: Minimal 4GB (disarankan 8GB+ untuk produksi)
- **CPU**: Minimal 2 vCPU
- **Storage**: Minimal 10GB (tergantung kebutuhan data)
- **Java**: OpenSearch membutuhkan **JDK 11 atau 17** (disarankan **OpenJDK 17**)

#### **2. Dependensi yang Dibutuhkan**
- **Java (OpenJDK 17)**
- **cURL atau Wget**
- **Unzip**
- **FirewallD (jika digunakan)**

---

### **Langkah Instalasi OpenSearch di Rocky Linux 8.10**

#### **1. Update dan Install Dependensi**
```bash
sudo dnf update -y
sudo dnf install -y java-17-openjdk wget unzip curl
```

#### **2. Download OpenSearch**
Versi terbaru OpenSearch bisa didapatkan dari [situs resminya](https://opensearch.org/downloads.html). Atau gunakan perintah berikut untuk mengunduh versi terbaru (misalnya 2.12.0):

```bash
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.12.0/opensearch-2.12.0-linux-x64.tar.gz
```

#### **3. Ekstrak File OpenSearch**
```bash
tar -xzf opensearch-2.12.0-linux-x64.tar.gz
cd opensearch-2.12.0
```

#### **4. Konfigurasi OpenSearch**
Buka file konfigurasi utama:
```bash
nano config/opensearch.yml
```
Sesuaikan beberapa pengaturan penting:
```yaml
network.host: 0.0.0.0  # Mengizinkan akses dari jaringan
discovery.type: single-node  # Untuk mode single-node
plugins.security.disabled: true  # Nonaktifkan keamanan untuk pengujian (jangan gunakan di produksi)
```
**(Opsional: Untuk produksi, gunakan konfigurasi cluster dan atur keamanan dengan TLS.)**

#### **5. Jalankan OpenSearch**
```bash
./opensearch-tar-install.sh
```
Atau secara manual:
```bash
./bin/opensearch
```
> Tunggu hingga OpenSearch aktif, biasanya berjalan di port **9200**.

#### **6. Uji OpenSearch**
Setelah OpenSearch berjalan, uji dengan:
```bash
curl -X GET "http://localhost:9200"
```
Jika berhasil, akan muncul respons JSON seperti ini:
```json
{
  "name" : "opensearch-node1",
  "cluster_name" : "opensearch",
  "version" : {
    "distribution" : "opensearch",
    "number" : "2.12.0",
    ...
  }
}
```

#### **7. (Opsional) Menjalankan sebagai Layanan Systemd**
Buat file service di `/etc/systemd/system/opensearch.service`:
```bash
sudo nano /etc/systemd/system/opensearch.service
```
Isi dengan:
```ini
[Unit]
Description=OpenSearch Server
After=network.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/path/to/opensearch/bin/opensearch
Restart=always
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```
> Ganti `/path/to/opensearch/` dengan lokasi OpenSearch yang sesuai.

Jalankan dan enable layanan:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opensearch
sudo systemctl status opensearch
```

#### **8. (Opsional) Membuka Firewall**
Jika ingin mengakses OpenSearch dari luar:
```bash
sudo firewall-cmd --add-port=9200/tcp --permanent
sudo firewall-cmd --reload
```

---

### **Langkah Berikutnya**
- Jika butuh **dashboard**, install **OpenSearch Dashboards** untuk visualisasi data.
- Untuk **klaster multi-node**, konfigurasikan beberapa instance dengan pengaturan discovery yang sesuai.
- Untuk **keamanan di produksi**, aktifkan TLS dan autentikasi.

Kamu mau deploy di single-node atau multi-node? 🔥