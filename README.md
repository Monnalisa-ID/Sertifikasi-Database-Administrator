# Sertifikasi Database Administrator
====================================================================
# Skema Prosedur Pengujian Backup
# Skema Backup dan Pemulihan Database untuk Sistem Manajemen Inventaris

Berikut adalah skema komprehensif untuk backup database termasuk penjadwalan harian dan prosedur pengujian pemulihan data:

## 1. Strategi Backup Multi-Lapis

**3-2-1 Backup Rule:**
- **3** salinan data (produksi + 2 backup)
- **2** media penyimpanan berbeda (local + cloud)
- **1** salinan di lokasi offsite

## 2. Skema Penjadwalan Backup

### Backup Harian (Full + Incremental)
```bash
#!/bin/bash
# backup_daily.sh
# Direktori backup
BACKUP_DIR="/var/backups/mysql"
DATE=$(date +%Y%m%d)
LOG_FILE="/var/log/mysql_backup.log"

# Buat direktori jika belum ada
mkdir -p $BACKUP_DIR/full
mkdir -p $BACKUP_DIR/incremental

# Backup full setiap Minggu
if [ $(date +%u) -eq 7 ]; then
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] Memulai backup full" >> $LOG_FILE
    mysqldump -u backup_user -p'password' --single-transaction --routines --triggers --all-databases | gzip > $BACKUP_DIR/full/[nama_database_backup].sql.gz
    
    # Rotasi backup (simpan 4 backup terakhir)
    ls -t $BACKUP_DIR/full/full_backup_* | tail -n +5 | xargs rm -f
else
    # Backup incremental harian
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] Memulai backup incremental" >> $LOG_FILE
    mysql -u backup_user -p'password' -e "FLUSH LOGS;"
    LAST_LOG=$(ls -t /var/lib/mysql/mysql-bin.?????? | head -1)
    cp $LAST_LOG $BACKUP_DIR/incremental/incr_backup_$DATE.binlog
    
    # Rotasi backup incremental (simpan 7 hari terakhir)
    ls -t $BACKUP_DIR/incremental/incr_backup_* | tail -n +8 | xargs rm -f
fi

# Sync ke cloud storage
echo "[$(date +'%Y-%m-%d %H:%M:%S')] Upload ke cloud storage" >> $LOG_FILE
rclone copy $BACKUP_DIR remote:backup-inventaris --log-file=$LOG_FILE --log-level INFO
```

### Penjadwalan dengan Cron
```bash
# Edit crontab
crontab -e

# Tambahkan jadwal berikut (backup setiap jam 2 dini hari)
0 2 * * * /bin/bash /path/to/backup_daily.sh
```

## 3. Prosedur Pemulihan Data

### Pemulihan dari Backup Full
```bash
# Dekompres file backup
gunzip < /var/backups/mysql/full/[nama_database_backup].sql.gz | mysql -u root -p
```

### Pemulihan Point-in-Time (dengan binlog)
```bash
# Pertama restore full backup
gunzip < [nama_database_backup].sql.gz | mysql -u root -p

# Kemudian apply incremental backup
mysqlbinlog /var/backups/mysql/incremental/incr_[nama_database_backup].binlog | mysql -u root -p

# Untuk recovery sampai waktu tertentu
mysqlbinlog --stop-datetime="2024-03-15 14:30:00" /var/backups/mysql/incremental/*.binlog | mysql -u root -p
```

## 4. Prosedur Pengujian Backup

### Skema Pengujian Bulanan
1. **Siapkan lingkungan testing**:
   ```bash
   # Buat database testing
   mysql -u root -p -e "CREATE DATABASE recovery_test;"
   ```

2. **Restore backup ke lingkungan testing**:
   ```bash
   gunzip < /var/backups/mysql/full/full_backup_$(date +%Y%m%d).sql.gz | mysql -u root -p recovery_test
   ```

3. **Validasi data**:
   ```sql
   -- Cek jumlah tabel
   SELECT COUNT(*) FROM information_schema.tables 
   WHERE table_schema = 'recovery_test';
   
   -- Cek data sample
   SELECT * FROM recovery_test.products LIMIT 5;
   SELECT COUNT(*) FROM recovery_test.transaction_details;
   ```

4. **Uji transaksi**:
   ```sql
   -- Lakukan beberapa operasi CRUD
   INSERT INTO recovery_test.categories (category_name) VALUES ('Test Category');
   UPDATE recovery_test.products SET price = price * 1.1 WHERE category_id = 1;
   DELETE FROM recovery_test.transactions WHERE transaction_id = 1;
   
   -- Verifikasi perubahan
   SELECT * FROM recovery_test.categories WHERE category_name LIKE 'Test%';
   ```

5. **Dokumentasi hasil**:
   - Catat waktu restore
   - Verifikasi kelengkapan data
   - Identifikasi masalah jika ada

## 5. Monitoring dan Alerting

### Skrip Monitoring Backup
```bash
#!/bin/bash
# check_backup.sh
LAST_BACKUP=$(find /var/backups/mysql/full -name "*.gz" -type f -printf '%T@ %p\n' | sort -n | tail -1 | cut -f2- -d" ")

# Cek jika backup lebih dari 24 jam
if [ -n "$LAST_BACKUP" ]; then
    BACKUP_AGE=$(( ( $(date +%s) - $(stat -c %Y "$LAST_BACKUP") ) / 3600 ))
    
    if [ $BACKUP_AGE -gt 24 ]; then
        echo "Peringatan: Backup terakhir berusia $BACKUP_AGE jam" | mail -s "Backup Warning" admin@example.com
    fi
else
    echo "Peringatan: Tidak ada backup ditemukan" | mail -s "Backup Warning" admin@example.com
fi
```

### Integrasi dengan Monitoring Tools
```yaml
# Contoh konfigurasi Prometheus
scrape_configs:
  - job_name: 'mysql_backup'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['backup-monitor:9100']
    params:
      backup_dir: ['/var/backups/mysql']
```

## 6. Dokumentasi Prosedur Backup

**Dokumen Resmi Backup Policy:**
1. **Frekuensi**:
   - Full backup: Mingguan (Minggu malam)
   - Incremental backup: Harian (kecuali hari Minggu)

2. **Retensi**:
   - Full backup: 4 minggu
   - Incremental backup: 7 hari
   - Cloud backup: 30 hari

3. **Tim Responsible**:
   - Primary: Database Administrator
   - Secondary: SysOps Engineer

4. **Prosedur Darurat**:
   ```markdown
   1. Identifikasi scope data yang hilang
   2. Pilih backup terdekat sebelum insiden
   3. Restore ke server staging
   4. Validasi data
   5. Restore ke produksi selama maintenance window
   6. Notifikasi stakeholder tentang data yang mungkin hilang
   ```

## 7. Otomatisasi dengan Tools Enterprise

Untuk lingkungan produksi kritis, pertimbangkan tools seperti:
- **Percona XtraBackup** untuk backup fisik MySQL tanpa downtime
- **BorgBackup** untuk backup terenkripsi dan terdeduplikasi
- **AWS Backup** untuk solusi cloud terkelola
- **Zabbix/Grafana** untuk monitoring backup

Skema ini memberikan pendekatan komprehensif untuk memastikan keamanan data dengan backup rutin dan prosedur pemulihan yang teruji.

# **Rencana Pemeliharaan Database Sistem Manajemen Inventaris**
**Periode:** 1 Januari 2025 - 31 Desember 2025

## **1. Tujuan Pemeliharaan**
- Memastikan ketersediaan database 24/7
- Mencegah kehilangan data
- Mengoptimalkan performa database
- Memenuhi standar keamanan data

---

## **2. Jadwal Pemeliharaan Rutin**

### **A. Pemeliharaan Harian**
**Waktu:** Setiap pukul 02.00 WIB (saat beban rendah)

| Aktivitas | Deskripsi | Tools/Perintah |
|-----------|-----------|----------------|
| **Backup Full Database** | Backup seluruh database | `mysqldump -u admin -p [nama_database] > /backup/daily/full_$(date +%Y%m%d).sql` |
| **Monitoring Resource** | Cek CPU, RAM, Disk Usage | `SHOW STATUS LIKE 'Threads_connected';`, `top`, `df -h` |
| **Log Cleaning** | Bersihkan log yang tidak perlu | `PURGE BINARY LOGS BEFORE '2025-01-01 00:00:00';` |

### **B. Pemeliharaan Mingguan**
**Waktu:** Setiap Minggu pukul 03.00 WIB

| Aktivitas | Deskripsi | Tools/Perintah |
|-----------|-----------|----------------|
| **Optimasi Tabel** | Defragmentasi tabel | `OPTIMIZE TABLE products, categories, transactions;` |
| **Backup Incremental** | Backup perubahan data sejak backup terakhir | `mysqlbinlog --start-datetime="2025-01-01 00:00:00" /var/log/mysql-bin.000123 > incremental_backup.sql` |
| **Cek Integrity Data** | Verifikasi konsistensi data | `CHECK TABLE products, categories FAST;` |

### **C. Pemeliharaan Bulanan**
**Waktu:** Hari Pertama setiap bulan pukul 04.00 WIB

| Aktivitas | Deskripsi | Tools/Perintah |
|-----------|-----------|----------------|
| **Update Database Schema** | Penyesuaian struktur tabel jika diperlukan | `ALTER TABLE products ADD COLUMN IF NOT EXISTS discount DECIMAL(5,2);` |
| **Audit Keamanan** | Review user privileges & aktivitas mencurigakan | `SELECT * FROM mysql.user;`, `SHOW GRANTS FOR 'admin'@'localhost';` |
| **Uji Pemulihan Backup** | Restore backup ke staging environment | `mysql -u admin -p staging_db < /backup/monthly/full_20250101.sql` |

### **D. Pemeliharaan Tahunan**
**Waktu:** 1 Januari 2026 (Evaluasi Tahunan)

| Aktivitas | Deskripsi |
|-----------|-----------|
| **Review Rencana Backup** | Evaluasi strategi backup (full/incremental) |
| **Upgrade MySQL Server** | Update ke versi stabil terbaru |
| **Pelatihan Admin Database** | Refresh pengetahuan tim IT |

---

## **3. Prosedur Pemulihan Darurat**
**Jika terjadi crash database:**
1. **Identifikasi Masalah**:
   - Cek error log: `tail -n 100 /var/log/mysql/error.log`
2. **Restore dari Backup Terakhir**:
   ```bash
   mysql -u admin -p [nama_database] < /backup/daily/full_20250101.sql
   ```
3. **Apply Incremental Backup** (jika ada):
   ```bash
   mysqlbinlog /backup/weekly/incremental_20250107.sql | mysql -u admin -p
   ```
4. **Verifikasi Data**:
   ```sql
   SELECT COUNT(*) FROM products;
   ```

---

## **4. Dokumentasi & Pelaporan**
- **Log Aktivitas**: Catat semua pemeliharaan di `maintenance_log.txt`
- **Laporan Bulanan**: Kirim laporan ke manajemen setiap akhir bulan

**Contoh Format Log:**
```
[2025-01-01 02:00] Backup harian selesai. Ukuran: 1.2GB
[2025-01-07 03:00] Optimasi tabel produk selesai. Waktu: 15 menit
```

---

## **5. Tim Responsible**
| Role | Nama | Kontak |
|------|------|--------|
| **Database Admin** | Mohammad Rizqi Aryanto | rizqiaryanto002@gmail.com |
| **SysOps Engineer** | Mohammad Rizqi Aryanto | rizqiaryanto002@gmail.com |

