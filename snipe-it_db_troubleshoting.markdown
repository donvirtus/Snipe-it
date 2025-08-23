# README: Troubleshooting Akses Database Snipe-IT dengan DBeaver

## Latar Belakang
Pada tanggal 23 Agustus 2025, kami mengalami masalah saat mencoba mengakses database `snipeit_db` menggunakan DBeaver dari mesin client (IP 30.1.1.7) ke server MariaDB (IP 20.1.1.16). Awalnya, koneksi ditolak dengan pesan error "Socket fail to connect... Connection refused," dan kemudian berubah menjadi "Host '30.1.1.7' is not allowed to connect to this MariaDB server" setelah beberapa perubahan konfigurasi.

## Langkah-langkah Troubleshooting

### 1. **Identifikasi Masalah Awal**
- **Gejala**: DBeaver tidak bisa connect ke MariaDB di 20.1.1.16:3306 dengan konfigurasi:
  - URL: `jdbc:mariadb://20.1.1.16:3306/snipeit_db`
  - Username: `snipeit_user`
  - Password: (tersimpan)
  - Error: "Socket fail to connect... Connection refused"
- **Analisis**: Ini menunjukkan masalah jaringan atau konfigurasi server, bukan kesalahan kredensial.

### 2. **Periksa Status Server MariaDB**
- Jalankan: `sudo systemctl status mariadb`
- Hasil: Server berjalan normal sejak 12:19:35 UTC, dengan status "Taking your SQL requests now..."
- Kesimpulan: Server aktif, tapi perlu cek konfigurasi jaringan.

### 3. **Periksa File Konfigurasi MariaDB (50-server.cnf)**
- File: `/etc/mysql/mariadb.conf.d/50-server.cnf`
- Perubahan: `bind-address` diubah dari `127.0.0.1` ke `0.0.0.0` untuk mengizinkan koneksi dari semua interface.
- Hasil: Konfigurasi benar, tetapi error berubah menjadi "Host not allowed," menunjukkan masalah privileges user.

### 4. **Periksa Privileges User**
- Jalankan: `SHOW GRANTS FOR 'snipeit_user'@'localhost';`
- Hasil Awal: User hanya diizinkan dari `localhost` dengan `GRANT ALL PRIVILEGES ON `snipeit_db`.* TO `snipeit_user`@`localhost``.
- Solusi: Tambah grant untuk remote access:
  ```
  GRANT ALL PRIVILEGES ON snipeit_db.* TO 'snipeit_user'@'%' IDENTIFIED BY PASSWORD '*001C5B099BBC0C504DC35E6268CEA9934A0C3993';
  FLUSH PRIVILEGES;
  ```
- Verifikasi: `SHOW GRANTS FOR 'snipeit_user'@'%';` menunjukkan grant baru aktif.

### 5. **Test Koneksi**
- Dari client (30.1.1.7): `mysql -h 20.1.1.16 -u snipeit_user -p -P 3306 snipeit_db`
- Hasil: Koneksi berhasil setelah grant diterapkan.
- DBeaver: Koneksi sukses, data tabel `assets` (49 record) terlihat.

## Penyebab Utama
- Awalnya, server MariaDB hanya bind ke `localhost`, dan user `snipeit_user` tidak diizinkan untuk koneksi remote. Perubahan `bind-address` membuka port 3306, tetapi privileges user perlu diperbarui untuk mengizinkan host 30.1.1.7.

## Solusi Akhir
- Ubah `bind-address` ke `0.0.0.0` di `50-server.cnf` dan restart MariaDB (`sudo systemctl restart mariadb`).
- Tambah grant remote untuk `snipeit_user` dengan `GRANT ... TO 'snipeit_user'@'%'` dan flush privileges.
- Pastikan firewall (misal UFW) mengizinkan port 3306 (`sudo ufw allow 3306/tcp`).

## Catatan Tambahan
- Perubahan privileges mungkin tidak langsung terlihat di sesi MariaDB yang sama; keluar dan login ulang diperlukan untuk refresh.
- Pastikan jaringan antar 20.1.1.16 dan 30.1.1.7 lancar (ping atau telnet 20.1.1.16:3306).
- Dokumentasi ini dibuat pada 08:38 PM WIB, 23 Agustus 2025.

## Langkah Selanjutnya
- Integrasi dengan script Python (setup_internusa_food.py) untuk pengelolaan data PT. Internusa Food.
- Prediksi harga crypto akan dilanjutkan setelah tugas ini selesai.