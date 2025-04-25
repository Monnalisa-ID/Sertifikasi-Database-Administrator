# Sertifikasi Database Administrator
 Serifikasi Database Administrator
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
| **Backup Full Database** | Backup seluruh database | `mysqldump -u admin -p manajemen_inventaris > /backup/daily/full_$(date +%Y%m%d).sql` |
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
   mysql -u admin -p manajemen_inventaris < /backup/daily/full_20250101.sql
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
=========================
# Skema Prosedur Pengujian Backup
===========================================================================================
# Siapkan lingkungan testing:
# - Buat database testing

mysql -u root -p -e "CREATE DATABASE recovery_test;"
===========================================================================================
# Restore backup ke lingkungan testing
gunzip < /var/backups/mysql/full/full_backup_$(date +%Y%m%d).sql.gz | mysql -u root -p recovery_test
===========================================================================================
# Validasi data:
-- Cek jumlah tabel
SELECT COUNT(*) FROM information_schema.tables
WHERE table_schema = 'recovery_test';

-- Cek data sample
SELECT * FROM recovery_test.products LIMIT 5;
SELECT COUNT(*) FROM recovery_test.transaction_details;
===========================================================================================
# Uji Transaksi :
-- Lakukan beberapa operasi CRUD
INSERT INTO recovery_test.categories (category_name) VALUES ('Test Category');
UPDATE recovery_test.products SET price = price * 1.1 WHERE category_id = 1;
DELETE FROM recovery_test.transactions WHERE transaction_id = 1;

-- Verifikasi perubahan
SELECT * FROM recovery_test.categories WHERE category_name LIKE 'Test%';
===========================================================================================
