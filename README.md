# 🏍️ Sistem Database Bengkel Motor Pro (MySQL)

> Project Database Relasional menggunakan **MySQL 8.4.3**
> Studi kasus: Manajemen Data Bengkel Motor dengan Automasi Trigger & View Laporan

---

## 👥 Anggota Kelompok

| No | Nama                |
| -- | ------------------- |
| 1  | Geraldy Febriansyah |
| 2  | Naufal Imania       |
| 3  | Aulia Novi          |

🔗 LinkedIn (Project Owner):
[https://www.linkedin.com/in/geraldy-febriansyah-b850473a6](https://www.linkedin.com/in/geraldy-febriansyah-b850473a6)

---

# 📌 Deskripsi Project

Project ini merupakan implementasi sistem database bengkel motor berbasis **MySQL** yang tidak hanya berfungsi sebagai penyimpanan data, tetapi juga memiliki **logika otomatis (Triggers)** dan **laporan dinamis (Views)**.

### 🎯 Tujuan

* Mengelola data pelanggan, motor, dan servis
* Menjaga konsistensi data secara otomatis
* Menyediakan laporan tanpa query kompleks

### ⚙️ Fitur Utama

| Fitur    | Keterangan                                         |
| -------- | -------------------------------------------------- |
| Triggers | 6 trigger untuk validasi, logging, dan otomatisasi |
| Views    | 2 view untuk laporan                               |
| Relasi   | Foreign Key untuk menjaga integritas data          |
| Audit    | Log perubahan & arsip data                         |

---

# 🗄️ 1. Setup Database & Tabel Tambahan

```sql
CREATE DATABASE db_bengkel_motor_pro;
USE db_bengkel_motor_pro;

CREATE TABLE log_pembayaran (
    id_log INT PRIMARY KEY AUTO_INCREMENT,
    id_servis INT,
    biaya_lama DECIMAL(10,2),
    biaya_baru DECIMAL(10,2),
    waktu_perubahan DATETIME
);

CREATE TABLE arsip_servis (
    id_arsip INT PRIMARY KEY AUTO_INCREMENT,
    id_motor INT,
    keluhan TEXT,
    tanggal_hapus DATETIME
);
```

---

# 📊 2. Struktur Tabel Utama

## 🔹 Tabel pelanggan

| Field          | Tipe Data    | Keterangan     |
| -------------- | ------------ | -------------- |
| id_pelanggan   | INT (PK, AI) | ID pelanggan   |
| nama_pelanggan | VARCHAR(100) | Nama pelanggan |
| alamat         | TEXT         | Alamat         |
| no_telp        | VARCHAR(15)  | Nomor telepon  |

```sql
CREATE TABLE pelanggan (
    id_pelanggan INT PRIMARY KEY AUTO_INCREMENT,
    nama_pelanggan VARCHAR(100),
    alamat TEXT,
    no_telp VARCHAR(15)
);
```

---

## 🔹 Tabel motor

| Field               | Tipe Data    | Keterangan          |
| ------------------- | ------------ | ------------------- |
| id_motor            | INT (PK, AI) | ID motor            |
| plat_nomor          | VARCHAR(15)  | Unik                |
| merk                | VARCHAR(50)  | Merk motor          |
| id_pelanggan        | INT (FK)     | Relasi ke pelanggan |
| total_servis_dibuat | INT          | Counter servis      |

```sql
CREATE TABLE motor (
    id_motor INT PRIMARY KEY AUTO_INCREMENT,
    plat_nomor VARCHAR(15) UNIQUE,
    merk VARCHAR(50),
    id_pelanggan INT,
    total_servis_dibuat INT DEFAULT 0,
    FOREIGN KEY (id_pelanggan) REFERENCES pelanggan(id_pelanggan)
);
```

---

## 🔹 Tabel servis

| Field             | Tipe Data     | Keterangan      |
| ----------------- | ------------- | --------------- |
| id_servis         | INT (PK, AI)  | ID servis       |
| id_motor          | INT (FK)      | Relasi ke motor |
| tanggal_servis    | DATE          | Tanggal         |
| keluhan           | TEXT          | Keluhan         |
| biaya             | DECIMAL(10,2) | Biaya           |
| status_pembayaran | VARCHAR(20)   | Status          |

```sql
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

# ⚡ 3. Implementasi Trigger

| No | Nama Trigger              | Event         | Fungsi          |
| -- | ------------------------- | ------------- | --------------- |
| 1  | trg_upper_plat            | BEFORE INSERT | Uppercase plat  |
| 2  | trg_validasi_biaya        | BEFORE INSERT | Validasi biaya  |
| 3  | trg_log_biaya             | AFTER UPDATE  | Log perubahan   |
| 4  | trg_update_counter_servis | AFTER INSERT  | Hitung servis   |
| 5  | trg_protect_lunas         | BEFORE UPDATE | Proteksi status |
| 6  | trg_arsip_servis          | AFTER DELETE  | Arsip data      |

---

### 1. Auto Uppercase Plat

```sql
CREATE TRIGGER trg_upper_plat
BEFORE INSERT ON motor
FOR EACH ROW
SET NEW.plat_nomor = UPPER(NEW.plat_nomor);
```

---

### 2. Validasi Biaya

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

---

### 3. Log Perubahan Biaya

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

---

### 4. Counter Servis Motor

```sql
CREATE TRIGGER trg_update_counter_servis
AFTER INSERT ON servis
FOR EACH ROW
UPDATE motor 
SET total_servis_dibuat = total_servis_dibuat + 1 
WHERE id_motor = NEW.id_motor;
```

---

### 5. Proteksi Status Lunas

```sql
DELIMITER //
CREATE TRIGGER trg_protect_lunas
BEFORE UPDATE ON servis
FOR EACH ROW
BEGIN
    IF OLD.status_pembayaran = 'lunas' 
       AND NEW.status_pembayaran = 'belum lunas' THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Transaksi lunas tidak bisa dibatalkan!';
    END IF;
END //
DELIMITER ;
```

---

### 6. Arsip Data Terhapus

```sql
CREATE TRIGGER trg_arsip_servis
AFTER DELETE ON servis
FOR EACH ROW
INSERT INTO arsip_servis(id_motor, keluhan, tanggal_hapus)
VALUES (OLD.id_motor, OLD.keluhan, NOW());
```

---

# 👁️ 4. Implementasi View

| No | Nama View            | Fungsi         |
| -- | -------------------- | -------------- |
| 1  | v_riwayat_lengkap    | Riwayat servis |
| 2  | v_pendapatan_bulanan | Laporan omzet  |

---

### 1. View Riwayat Lengkap

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

---

### 2. View Pendapatan Bulanan

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

# 📝 5. Contoh Hasil Output

## 🔹 Riwayat Servis

| nama_pelanggan | plat_nomor | merk  | tanggal_servis | keluhan   | biaya | status |
| -------------- | ---------- | ----- | -------------- | --------- | ----- | ------ |
| Geraldy        | B1234XYZ   | Honda | 2026-04-01     | Ganti oli | 75000 | lunas  |

---

## 🔹 Pendapatan Bulanan

| bulan   | total_transaksi | total_pendapatan |
| ------- | --------------- | ---------------- |
| 2026-04 | 5               | 350000           |

---

# 🧪 6. Pengujian

### Query View

```sql
SELECT * FROM v_riwayat_lengkap;
SELECT * FROM v_pendapatan_bulanan;
```

### Uji Validasi Trigger

```sql
INSERT INTO servis (id_motor, tanggal_servis, keluhan, biaya) 
VALUES (1, '2026-04-08', 'cek mesin', -50000);
```

---

# 🎯 Kesimpulan

| Aspek         | Hasil                            |
| ------------- | -------------------------------- |
| Keamanan Data | Validasi & proteksi otomatis     |
| Audit         | Tercatat log & arsip             |
| Efisiensi     | View menggantikan query kompleks |

Database ini sudah siap digunakan sebagai backend dan dapat diintegrasikan ke aplikasi seperti Laravel atau sistem manajemen bengkel.

---

# 🚀 Status Project

| Status      | Keterangan     |
| ----------- | -------------- |
| Level       | Advanced       |
| Dokumentasi | Lengkap        |
| Kesiapan    | Siap Integrasi |
