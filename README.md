# 🏍️ Sistem Database Bengkel Motor Pro (MySQL)

> Project Database Relasional menggunakan **MySQL 8.4.3**
> Studi kasus: Manajemen Data Bengkel Motor dengan fokus pada **Trigger** dan **View**

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

Project ini membangun sistem database bengkel motor yang menekankan pada:

* ✅ **Trigger** → untuk otomatisasi dan validasi data
* ✅ **View** → untuk menampilkan laporan tanpa query kompleks
* ✅ **Relasi Tabel** → menjaga integritas data antar tabel

---

# 🗄️ 1. Setup Database

```sql
CREATE DATABASE db_bengkel_motor_pro;
USE db_bengkel_motor_pro;
```

---

# 📊 2. Struktur Tabel

## 👤 Tabel `pelanggan`

| Field          | Tipe Data    | Keterangan   |
| -------------- | ------------ | ------------ |
| id_pelanggan   | INT (PK, AI) | ID pelanggan |
| nama_pelanggan | VARCHAR(100) | Nama         |
| alamat         | TEXT         | Alamat       |
| no_telp        | VARCHAR(15)  | Telepon      |

```sql
CREATE TABLE pelanggan (
    id_pelanggan INT PRIMARY KEY AUTO_INCREMENT,
    nama_pelanggan VARCHAR(100),
    alamat TEXT,
    no_telp VARCHAR(15)
);
```

---

## 🏍️ Tabel `motor`

| Field               | Tipe Data    | Keterangan          |
| ------------------- | ------------ | ------------------- |
| id_motor            | INT (PK, AI) | ID motor            |
| plat_nomor          | VARCHAR(15)  | Unik                |
| merk                | VARCHAR(50)  | Merk                |
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

## 🔧 Tabel `servis`

| Field             | Tipe Data     | Keterangan           |
| ----------------- | ------------- | -------------------- |
| id_servis         | INT (PK, AI)  | ID servis            |
| id_motor          | INT (FK)      | Relasi ke motor      |
| tanggal_servis    | DATE          | Tanggal              |
| keluhan           | TEXT          | Keluhan              |
| biaya             | DECIMAL(10,2) | Biaya                |
| status_pembayaran | VARCHAR(20)   | Default: belum lunas |

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

## 📁 Tabel Tambahan

### Log Pembayaran

```sql
CREATE TABLE log_pembayaran (
    id_log INT PRIMARY KEY AUTO_INCREMENT,
    id_servis INT,
    biaya_lama DECIMAL(10,2),
    biaya_baru DECIMAL(10,2),
    waktu_perubahan DATETIME
);
```

### Arsip Servis

```sql
CREATE TABLE arsip_servis (
    id_arsip INT PRIMARY KEY AUTO_INCREMENT,
    id_motor INT,
    keluhan TEXT,
    tanggal_hapus DATETIME
);
```

---

# 🔗 Relasi Database

| Relasi            | Penjelasan                             |
| ----------------- | -------------------------------------- |
| Pelanggan → Motor | 1 pelanggan bisa memiliki banyak motor |
| Motor → Servis    | 1 motor bisa memiliki banyak servis    |

---

# 📝 3. Insert Data (Contoh)

```sql
INSERT INTO pelanggan (nama_pelanggan, no_telp, alamat) VALUES
('geraldy', '0822', 'bandung');

INSERT INTO motor (plat_nomor, merk, id_pelanggan) VALUES
('b1234xyz', 'honda', 1);

INSERT INTO servis (id_motor, tanggal_servis, keluhan, biaya) VALUES
(1, '2026-04-01', 'ganti oli', 75000);
```

---

# ⚡ 4. Implementasi Trigger

## 1. Auto Uppercase Plat

```sql
CREATE TRIGGER trg_upper_plat
BEFORE INSERT ON motor
FOR EACH ROW
SET NEW.plat_nomor = UPPER(NEW.plat_nomor);
```

---

## 2. Validasi Biaya

```sql
DELIMITER //
CREATE TRIGGER trg_validasi_biaya
BEFORE INSERT ON servis
FOR EACH ROW
BEGIN
    IF NEW.biaya < 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Biaya tidak boleh negatif!';
    END IF;
END //
DELIMITER ;
```

---

## 3. Log Perubahan Biaya

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

## 4. Counter Servis

```sql
CREATE TRIGGER trg_update_counter_servis
AFTER INSERT ON servis
FOR EACH ROW
UPDATE motor 
SET total_servis_dibuat = total_servis_dibuat + 1
WHERE id_motor = NEW.id_motor;
```

---

## 5. Proteksi Status Lunas

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

## 6. Arsip Data Terhapus

```sql
CREATE TRIGGER trg_arsip_servis
AFTER DELETE ON servis
FOR EACH ROW
INSERT INTO arsip_servis(id_motor, keluhan, tanggal_hapus)
VALUES (OLD.id_motor, OLD.keluhan, NOW());
```

---

# 👁️ 5. Implementasi View

## 1. View Riwayat Servis

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

## 2. View Pendapatan Bulanan

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

# 🧪 6. Pengujian

## Menampilkan Data View

```sql
SELECT * FROM v_riwayat_lengkap;
SELECT * FROM v_pendapatan_bulanan;
```

---

## Contoh Hasil

### Riwayat Servis

| nama_pelanggan | plat_nomor | merk  | tanggal_servis | keluhan   | biaya | status_pembayaran |
| -------------- | ---------- | ----- | -------------- | --------- | ----- | ----------------- |
| geraldy        | B1234XYZ   | honda | 2026-04-01     | ganti oli | 75000 | belum lunas       |

---

### Pendapatan Bulanan

| bulan   | total_transaksi | total_pendapatan |
| ------- | --------------- | ---------------- |
| 2026-04 | 1               | 75000            |

---

## Uji Trigger Validasi

```sql
INSERT INTO servis (id_motor, tanggal_servis, keluhan, biaya)
VALUES (1, '2026-04-02', 'cek mesin', -50000);
```

---

# 🎯 Kesimpulan

| Aspek   | Hasil                                 |
| ------- | ------------------------------------- |
| Trigger | Berjalan untuk validasi & otomatisasi |
| View    | Menyederhanakan laporan               |
| Relasi  | Data terhubung dengan baik            |

Database ini menunjukkan bagaimana **Trigger** dan **View** dapat meningkatkan efisiensi dan keamanan dalam sistem database.

---

# 🚀 Status Project

| Status   | Keterangan      |
| -------- | --------------- |
| Struktur | Selesai         |
| Trigger  | Aktif           |
| View     | Berjalan        |
| Kesiapan | Siap presentasi |
