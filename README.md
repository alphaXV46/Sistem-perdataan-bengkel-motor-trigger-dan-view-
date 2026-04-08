# 🏍️ Sistem Database Bengkel Motor Pro (MySQL)

> Project Database Relasional menggunakan **MySQL 8.4.3**
> Studi kasus: Manajemen Data Bengkel Motor dengan Automasi Trigger & View Laporan.

---

## 👥 Anggota Kelompok

1. **Geraldy Febriansyah**
2. **Naufal Imania**
3. **Aulia Novi**

🔗 LinkedIn (Project Owner):
[https://www.linkedin.com/in/geraldy-febriansyah-b850473a6](https://www.linkedin.com/in/geraldy-febriansyah-b850473a6)

---

# 📌 Deskripsi Project

Project ini mengimplementasikan sistem database bengkel yang tidak hanya menyimpan data, tetapi juga memiliki logika bisnis otomatis menggunakan **Triggers** untuk validasi data dan **Views** untuk penyajian laporan yang kompleks.

Fitur Utama:
* ✅ **6 Triggers:** Automasi validasi, logging, dan sinkronisasi data.
* ✅ **2 Views:** Laporan pendapatan bulanan dan riwayat servis lengkap.
* ✅ **Relational Integrity:** Penggunaan FK yang ketat.

---

# 🗄️ 1. Setup Database & Tabel Tambahan

Selain tabel utama, kita memerlukan tabel `log_aktivitas` dan `arsip_servis` untuk mendukung fungsi trigger.

```sql
CREATE DATABASE db_bengkel_motor_pro;
USE db_bengkel_motor_pro;

-- Tabel Log untuk memantau perubahan biaya servis
CREATE TABLE log_pembayaran (
    id_log INT PRIMARY KEY AUTO_INCREMENT,
    id_servis INT,
    biaya_lama DECIMAL(10,2),
    biaya_baru DECIMAL(10,2),
    waktu_perubahan DATETIME
);

-- Tabel Arsip untuk data servis yang dihapus
CREATE TABLE arsip_servis (
    id_arsip INT PRIMARY KEY AUTO_INCREMENT,
    id_motor INT,
    keluhan TEXT,
    tanggal_hapus DATETIME
);
```

---

# 📊 2. Struktur Tabel Utama

*(Struktur `pelanggan`, `motor`, dan `servis` tetap sama dengan modifikasi kolom audit jika diperlukan)*

```sql
CREATE TABLE pelanggan (
    id_pelanggan INT PRIMARY KEY AUTO_INCREMENT,
    nama_pelanggan VARCHAR(100),
    alamat TEXT,
    no_telp VARCHAR(15)
);

CREATE TABLE motor (
    id_motor INT PRIMARY KEY AUTO_INCREMENT,
    plat_nomor VARCHAR(15) UNIQUE,
    merk VARCHAR(50),
    id_pelanggan INT,
    total_servis_dibuat INT DEFAULT 0, -- Untuk Trigger hitung otomatis
    FOREIGN KEY (id_pelanggan) REFERENCES pelanggan(id_pelanggan)
);

CREATE TABLE servis (
    id_servis INT PRIMARY KEY AUTO_INCREMENT,
    id_motor INT,
    tanggal_servis DATE,
    keluhan TEXT,
    biaya DECIMAL(10,2),
    status_pembayaran VARCHAR(20) DEFAULT 'belum lunas',
    FOREIGN KEY (id_motor) REFERENCES motor(id_motor)
);
```

---

# ⚡ 3. Implementasi 6 Trigger

Trigger digunakan untuk memastikan data konsisten tanpa perlu campur tangan manual di sisi aplikasi.

### 1. Auto-Uppercase Plat Nomor (Before Insert)
Memastikan plat nomor selalu tersimpan dalam huruf kapital.
```sql
CREATE TRIGGER trg_upper_plat
BEFORE INSERT ON motor
FOR EACH ROW
SET NEW.plat_nomor = UPPER(NEW.plat_nomor);
```

### 2. Validasi Biaya Minimum (Before Insert)
Mencegah input biaya servis bernilai negatif.
```sql
DELIMITER //
CREATE TRIGGER trg_validasi_biaya
BEFORE INSERT ON servis
FOR EACH ROW
BEGIN
    IF NEW.biaya < 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Biaya tidak boleh negatif!';
    END IF;
END //
DELIMITER ;
```

### 3. Log Perubahan Biaya (After Update)
Mencatat riwayat jika ada perubahan nominal biaya pada tabel servis.
```sql
CREATE TRIGGER trg_log_biaya
AFTER UPDATE ON servis
FOR EACH ROW
BEGIN
    IF OLD.biaya <> NEW.biaya THEN
        INSERT INTO log_pembayaran(id_servis, biaya_lama, biaya_baru, waktu_perubahan)
        VALUES (OLD.id_servis, OLD.biaya, NEW.biaya, NOW());
    END IF;
END;
```

### 4. Counter Servis Motor (After Insert)
Menambah jumlah total servis pada tabel `motor` setiap kali ada transaksi servis baru.
```sql
CREATE TRIGGER trg_update_counter_servis
AFTER INSERT ON servis
FOR EACH ROW
UPDATE motor SET total_servis_dibuat = total_servis_dibuat + 1 
WHERE id_motor = NEW.id_motor;
```

### 5. Proteksi Status Lunas (Before Update)
Mencegah status yang sudah 'lunas' diubah kembali menjadi 'belum lunas'.
```sql
DELIMITER //
CREATE TRIGGER trg_protect_lunas
BEFORE UPDATE ON servis
FOR EACH ROW
BEGIN
    IF OLD.status_pembayaran = 'lunas' AND NEW.status_pembayaran = 'belum lunas' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Transaksi lunas tidak bisa dibatalkan!';
    END IF;
END //
DELIMITER ;
```

### 6. Arsip Data Terhapus (After Delete)
Memindahkan data servis yang dihapus ke tabel arsip agar tidak hilang sepenuhnya.
```sql
CREATE TRIGGER trg_arsip_servis
AFTER DELETE ON servis
FOR EACH ROW
INSERT INTO arsip_servis(id_motor, keluhan, tanggal_hapus)
VALUES (OLD.id_motor, OLD.keluhan, NOW());
```

---

# 👁️ 4. Implementasi 2 View

View memudahkan kita untuk melihat laporan tanpa perlu menulis query JOIN yang panjang berkali-kali.

### 1. View Riwayat Servis Lengkap
Menampilkan data gabungan pelanggan, motor, dan detail servisnya.
```sql
CREATE VIEW v_riwayat_lengkap AS
SELECT 
    p.nama_pelanggan,
    m.plat_nomor,
    m.merk,
    s.tanggal_servis,
    s.keluhan,
    s.biaya,
    s.status_pembayaran
FROM pelanggan p
JOIN motor m ON p.id_pelanggan = m.id_pelanggan
JOIN servis s ON m.id_motor = s.id_motor;
```

### 2. View Laporan Pendapatan Bulanan
Menghitung total uang masuk yang sudah lunas per bulan.
```sql
CREATE VIEW v_pendapatan_bulanan AS
SELECT 
    DATE_FORMAT(tanggal_servis, '%Y-%m') AS bulan,
    COUNT(id_servis) AS total_transaksi,
    SUM(biaya) AS total_pendapatan
FROM servis
WHERE status_pembayaran = 'lunas'
GROUP BY bulan;
```

---

# 📝 5. Uji Coba Query

### Menampilkan Laporan dari View
```sql
-- Melihat riwayat pelanggan
SELECT * FROM v_riwayat_lengkap WHERE nama_pelanggan = 'geraldy';

-- Melihat omzet bengkel
SELECT * FROM v_pendapatan_bulanan;
```

### Menguji Trigger Validasi
```sql
-- Ini akan error karena biaya negatif
INSERT INTO servis (id_motor, tanggal_servis, keluhan, biaya) 
VALUES (1, '2026-04-08', 'cek mesin', -50000);
```

---

# 🎯 Kesimpulan & Pengembangan

Dengan penambahan **Triggers** dan **Views**, database ini sekarang memiliki:
1. **Keamanan Data:** Validasi nilai dan proteksi status pembayaran.
2. **Audit Trail:** Riwayat perubahan biaya dan arsip data terhapus.
3. **Efisiensi:** Laporan dapat diakses langsung melalui View tanpa JOIN manual.

Database ini siap diintegrasikan dengan aplikasi **Laravel** (Project DefaCraftStore kamu) atau sistem manajemen internal bengkel lainnya.

---

# 🚀 Status Project
✅ **Level: Advanced Backend Ready**
✅ Dokumentasi Lengkap
✅ Logic Teruji