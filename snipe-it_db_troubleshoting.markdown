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

## Pembatasan Akses ke IP atau MAC Address Tertentu
Untuk meningkatkan keamanan, akses dapat dibatasi hanya untuk IP atau MAC address tertentu:

### Membatasi Berdasarkan IP
- MariaDB mendukung pembatasan berdasarkan IP di level user privileges.
- **Langkah:**
  1. Login ke MariaDB: `sudo mysql -u root -p`.
  2. (Opsional) Hapus grant lama untuk '%':
     ```
     REVOKE ALL PRIVILEGES ON snipeit_db.* FROM 'snipeit_user'@'%';
     DROP USER 'snipeit_user'@'%';
     FLUSH PRIVILEGES;
     ```
  3. Tambah grant untuk IP spesifik (misal 30.1.1.7 dan 30.1.1.8):
     ```
     CREATE USER 'snipeit_user'@'30.1.1.7' IDENTIFIED BY 'password_kamu';
     GRANT ALL PRIVILEGES ON snipeit_db.* TO 'snipeit_user'@'30.1.1.7';
     CREATE USER 'snipeit_user'@'30.1.1.8' IDENTIFIED BY 'password_kamu';
     GRANT ALL PRIVILEGES ON snipeit_db.* TO 'snipeit_user'@'30.1.1.8';
     FLUSH PRIVILEGES;
     ```
     (Ganti 'password_kamu' dengan password asli.)
  4. Verifikasi: `SHOW GRANTS FOR 'snipeit_user'@'30.1.1.7';`.
  5. Test: Koneksi dari IP lain (misal 30.1.1.9) akan ditolak.
- **Catatan**: Gunakan IP spesifik sesuai kebutuhan PT. Internusa Food untuk keamanan lebih baik daripada '%'.

### Membatasi Berdasarkan MAC Address
- MariaDB tidak langsung mendukung filter MAC, karena MAC hanya relevan di layer data link dan tidak terlihat di koneksi TCP/IP remote. Pendekatan ini memerlukan konfigurasi firewall:
  - **Langkah:**
    1. Cek MAC address client: `arp -n` di server setelah ping dari client.
    2. Tambah rule iptables: `sudo iptables -A INPUT -p tcp -m mac --mac-source 00:14:22:01:23:45 -d 20.1.1.16 --dport 3306 -j ACCEPT`.
    3. Blok koneksi lain: `sudo iptables -A INPUT -p tcp --dport 3306 -j DROP`.
  - **Keterbat
