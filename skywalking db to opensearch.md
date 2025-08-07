Untuk menyambungkan **Apache SkyWalking** ke **OpenSearch** sebagai backend storage, kita perlu mengonfigurasi **storage** di SkyWalking agar menggunakan OpenSearch. Berikut langkah-langkahnya:

---

## **1. Persiapan**
### **a. Pastikan OpenSearch Sudah Berjalan**
Cek apakah OpenSearch sudah berjalan dengan:
```sh
curl -X GET "http://localhost:9200/_cluster/health?pretty"
```
Jika berjalan, akan muncul output dengan status **green** atau **yellow**.

---

## **2. Konfigurasi SkyWalking ke OpenSearch**
SkyWalking menggunakan konfigurasi berbasis `application.yml` yang berada di dalam direktori **config**.  

### **a. Edit File Konfigurasi**
Buka file konfigurasi:
```sh
nano /home/admin1/apache-skywalking-apm-bin/config/application.yml
```

Cari bagian **storage** dan ubah menjadi seperti berikut:

```yaml
storage:
  selector: ${SW_STORAGE:opensearch}
  opensearch:
    namespace: ${SW_NAMESPACE:"skywalking"}
    clusterNodes: ${SW_STORAGE_OPENSEARCH_CLUSTER_NODES:localhost:9200}
    user: ${SW_STORAGE_OPENSEARCH_USER:""}
    password: ${SW_STORAGE_OPENSEARCH_PASSWORD:""}
    trustStorePath: ${SW_SW_STORAGE_ES_SSL_JKS_PATH:""}
    trustStorePass: ${SW_SW_STORAGE_ES_SSL_JKS_PASS:""}
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1}
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:1}
    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5}
```

### **b. Penjelasan Konfigurasi**
- **`selector: opensearch`** → Menentukan SkyWalking menggunakan OpenSearch sebagai storage.
- **`clusterNodes: localhost:9200`** → Ganti dengan alamat OpenSearch jika berada di server lain.
- **`user` & `password`** → Jika OpenSearch menggunakan autentikasi, masukkan username & password.
- **`indexShardsNumber` & `indexReplicasNumber`** → Sesuaikan dengan konfigurasi OpenSearch.

---

## **3. Restart SkyWalking**
Setelah mengedit file, restart SkyWalking:
```sh
sudo systemctl restart skywalking
```

Cek apakah SkyWalking berhasil tersambung ke OpenSearch dengan:
```sh
journalctl -u skywalking -f
```

---

## **4. Verifikasi Data di OpenSearch**
Cek apakah SkyWalking berhasil membuat indeks di OpenSearch:
```sh
curl -X GET "http://localhost:9200/_cat/indices?v"
```
Jika konfigurasi berhasil, akan muncul indeks seperti:
```
green open skywalking-segment ...
green open skywalking-metrics ...
green open skywalking-trace ...
```

---

Sekarang SkyWalking sudah menggunakan **OpenSearch sebagai backend storage**! 🚀 Jika ada error, bisa cek log SkyWalking dan OpenSearch untuk debugging lebih lanjut.
