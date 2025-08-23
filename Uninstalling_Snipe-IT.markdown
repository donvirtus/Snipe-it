# Panduan Lengkap Uninstalasi Snipe-IT (LEMP Stack) di Ubuntu

Dokumen ini menyediakan panduan langkah demi langkah untuk menghapus instalasi Snipe-IT secara total beserta komponen pendukungnya (Nginx, MariaDB, PHP) dari server Ubuntu. Prosedur ini akan mengembalikan server ke kondisi sebelum instalasi.

## Peringatan Penting
- **Prosedur di bawah ini, khususnya Langkah 3, bersifat destruktif** dan akan menghapus perangkat lunak dari server Anda.
- **Hanya untuk Uninstalasi Total**: Ikuti Langkah 3 hanya jika Anda tidak lagi memerlukan Nginx, MariaDB, atau PHP untuk aplikasi lain di server ini.
- **Backup Data**: Pastikan Anda telah mencadangkan data penting lainnya sebelum melanjutkan.

## Langkah 1: Hapus Aplikasi Snipe-IT & Konfigurasi Web Server (Nginx)
Langkah ini menghapus file aplikasi Snipe-IT dan menonaktifkan konfigurasinya di Nginx.

### Hapus Konfigurasi Nginx untuk Snipe-IT
Nonaktifkan situs Snipe-IT dengan menghapus symbolic link, kemudian hapus file konfigurasinya.

```bash
sudo rm /etc/nginx/sites-enabled/snipe-it
sudo rm /etc/nginx/sites-available/snipe-it
```

### (Opsional) Aktifkan Kembali Situs Default Nginx
Jika Anda ingin Nginx tetap berjalan, aktifkan kembali konfigurasi default.

```bash
sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
```

### Tes dan Restart Nginx
Pastikan konfigurasi Nginx valid sebelum me-restart layanan.

```bash
sudo nginx -t
sudo systemctl restart nginx
```

### Hapus Direktori Aplikasi Snipe-IT
Perintah ini akan menghapus semua file sumber kode Snipe-IT secara permanen.

```bash
sudo rm -rf /var/www/snipe-it
```

### Verifikasi Langkah 1
Pastikan direktori aplikasi sudah terhapus:

```bash
ls -l /var/www/
```
(Direktori `snipe-it` seharusnya sudah tidak ada).

Pastikan file konfigurasi Nginx sudah terhapus:

```bash
ls -l /etc/nginx/sites-available/
```
(File `snipe-it` seharusnya sudah tidak ada).

## Langkah 2: Hapus Database & User MariaDB
Langkah ini akan menghapus database `snipeit_db` dan pengguna `snipeit_user` yang dibuat saat instalasi.

### Masuk ke MariaDB sebagai root

```bash
sudo mysql -u root -p
```

### Jalankan Perintah SQL untuk Menghapus Database & User

```sql
DROP DATABASE snipeit_db;
DROP USER 'snipeit_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Verifikasi Langkah 2
Masuk kembali ke `sudo mysql -u root -p`.

Cek apakah database sudah tidak ada:

```sql
SHOW DATABASES;
```
(Pastikan `snipeit_db` tidak muncul dalam daftar).

Cek apakah pengguna sudah tidak ada:

```sql
SELECT user, host FROM mysql.user;
```
(Pastikan `snipeit_user` tidak muncul dalam daftar).

## Langkah 3: (Opsional) Uninstalasi Komponen Inti & Alat Bantu
**PERINGATAN**: Lanjutkan hanya jika Anda yakin server ini tidak lagi memerlukan Nginx, MariaDB, PHP, atau Composer untuk aplikasi lain.

### Uninstall Nginx

```bash
sudo systemctl stop nginx
sudo apt-get purge -y nginx nginx-common
```

### Uninstall MariaDB

```bash
sudo systemctl stop mariadb
sudo apt-get purge -y mariadb-server
sudo rm -rf /var/lib/mysql
```

### Uninstall PHP 8.2 & Ekstensinya

```bash
sudo apt-get purge -y php8.2-fpm php8.2-cli php8.2-mysql php8.2-gd php8.2-ldap php8.2-zip php8.2-mbstring php8.2-curl php8.2-xml php8.2-bcmath
```

### Hapus PPA PHP (Opsional)
Jika Anda tidak lagi memerlukan PPA dari Ondřej Surý.

```bash
sudo add-apt-repository --remove ppa:ondrej/php -y
```

### Uninstall Composer

```bash
sudo rm /usr/local/bin/composer
```

### Bersihkan Dependensi yang Tidak Terpakai
Perintah ini akan menghapus paket-paket lain yang diinstal sebagai dependensi dan tidak lagi diperlukan.

```bash
sudo apt-get autoremove -y
```

### Verifikasi Langkah 3
Cek status layanan (seharusnya gagal atau tidak ditemukan):

```bash
sudo systemctl status nginx
sudo systemctl status mariadb
```

Cek versi program (seharusnya menampilkan "command not found"):

```bash
nginx -v
mysql --version
php -v
composer --version
```

## Troubleshooting
### Error: "Read-only file system" saat Menghapus Database
**Penyebab**: Sistem mendeteksi error pada disk (kartu SD) dan mengaktifkan mode hanya-baca untuk proteksi.

**Solusi**:

Reboot server untuk memicu pengecekan disk otomatis:

```bash
sudo reboot
```

Jika tidak berhasil, paksa pengecekan disk pada boot berikutnya:

```bash
sudo touch /forcefsck
sudo reboot
```

Setelah server kembali online, coba ulangi langkah penghapusan database.