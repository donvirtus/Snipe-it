# Panduan Lengkap Instalasi Snipe-IT di Ubuntu Server

Dokumen ini menyediakan panduan langkah demi langkah untuk menginstal **Snipe-IT** pada server Ubuntu (22.04/24.04) menggunakan **Nginx**, **MariaDB**, dan **PHP** (LEMP Stack). Panduan ini mencakup solusi untuk masalah umum yang mungkin muncul selama instalasi.

## Prasyarat

Sebelum memulai, pastikan Anda memiliki:

- Server dengan Ubuntu 22.04 atau 24.04.
- Akses terminal dengan hak **sudo**.
- Koneksi internet yang stabil.

## Langkah 1: Persiapan Server (Instalasi LEMP Stack & Alat Bantu)

Kita akan menginstal komponen dasar yang dibutuhkan Snipe-IT.

### 1.1 Update Sistem

Perbarui daftar paket dan sistem untuk memastikan semua perangkat lunak mutakhir.

```bash
sudo apt update
sudo apt upgrade -y
```

### 1.2 Instal Nginx

Nginx akan digunakan sebagai web server.

```bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 1.3 Instal MariaDB (Database)

MariaDB akan menyimpan data aset Snipe-IT.

```bash
sudo apt install -y mariadb-server
```

**Catatan**: Pada versi Ubuntu baru, layanan MariaDB biasanya otomatis berjalan dan aktif setelah instalasi.

Jalankan skrip keamanan untuk mengatur kata sandi root dan mengamankan database.

```bash
sudo mysql_secure_installation
```

Ikuti petunjuk di layar untuk mengatur kata sandi root dan opsi keamanan lainnya.

### 1.4 Instal PHP 8.2 & Ekstensinya

Snipe-IT membutuhkan PHP 8.2. Kita akan menggunakan PPA dari Ondřej Surý untuk mendapatkan versi yang diperlukan.

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install -y php8.2-fpm php8.2-cli php8.2-mysql php8.2-gd php8.2-ldap php8.2-zip php8.2-mbstring php8.2-curl php8.2-xml php8.2-bcmath
```

#### 1.4.1 PENTING: Atur PHP 8.2 sebagai Versi Default

Jika sistem memiliki beberapa versi PHP (misalnya, 8.1 dan 8.2), Composer mungkin menggunakan versi lama. Atur PHP 8.2 sebagai default.

```bash
sudo update-alternatives --config php
```

Pilih nomor yang sesuai untuk `/usr/bin/php8.2` dan tekan Enter.

### 1.5 Instal Composer

Composer adalah manajer dependensi PHP untuk mengunduh library yang dibutuhkan Snipe-IT.

```bash
sudo apt install -y curl git unzip
curl -sS https://getcomposer.org/installer -o /tmp/composer-setup.php
sudo php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

## Langkah 2: Konfigurasi Database

Buat database dan pengguna khusus untuk Snipe-IT.

1. Masuk ke MariaDB sebagai root:

```bash
sudo mysql -u root -p
```

2. Jalankan perintah SQL berikut (ganti `password_yang_kuat` dengan kata sandi pilihan Anda):

```sql
CREATE DATABASE snipeit_db;
CREATE USER 'snipeit_user'@'localhost' IDENTIFIED BY 'password_yang_kuat';
GRANT ALL PRIVILEGES ON snipeit_db.* TO 'snipeit_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Langkah 3: Download dan Konfigurasi Snipe-IT

Unduh dan siapkan kode Snipe-IT.

1. Unduh Snipe-IT dari GitHub:

```bash
sudo git clone https://github.com/snipe/snipe-it /var/www/snipe-it
```

2. Pindah ke direktori Snipe-IT dan salin file konfigurasi:

```bash
cd /var/www/snipe-it
sudo cp .env.example .env
```

3. Edit file `.env` dengan editor teks:

```bash
sudo nano .env
```

Ubah baris berikut (gunakan alamat IP server jika diakses dari jaringan):

```env
APP_URL=http://localhost
APP_TIMEZONE='Asia/Jakarta'
DB_DATABASE=snipeit_db
DB_USERNAME=snipeit_user
DB_PASSWORD=password_yang_kuat
```

Simpan dan tutup file (Ctrl+X, Y, Enter).

4. Atur izin folder:

```bash
sudo chown -R www-data:www-data /var/www/snipe-it
sudo chmod -R 775 /var/www/snipe-it/storage
sudo chmod -R 775 /var/www/snipe-it/public/uploads
```

5. Instal dependensi dengan Composer:

```bash
sudo mkdir -p /var/www/.config/composer
sudo mkdir -p /var/www/.cache/composer
sudo chown -R www-data:www-data /var/www/.config
sudo chown -R www-data:www-data /var/www/.cache
sudo -u www-data composer install --no-dev --prefer-source
```

**Catatan**: Jika diminta GitHub Token, ikuti tautan yang muncul di terminal untuk membuat token (tanpa scope), lalu masukkan ke terminal.

6. Buat kunci aplikasi:

```bash
sudo php artisan key:generate
```

## Langkah 4: Konfigurasi Web Server (Nginx)

Konfigurasikan Nginx untuk menyajikan aplikasi Snipe-IT.

1. Buat file konfigurasi Nginx:

```bash
sudo nano /etc/nginx/sites-available/snipe-it
```

2. Salin konfigurasi berikut:

```nginx
server {
    listen 80;
    server_name localhost; # Ganti dengan IP atau domain Anda jika perlu

    root /var/www/snipe-it/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    }
}
```

3. Aktifkan konfigurasi:

```bash
sudo ln -s /etc/nginx/sites-available/snipe-it /etc/nginx/sites-enabled/
```

4. Hapus konfigurasi default untuk menghindari konflik:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

5. Pastikan file `nginx.conf` mengaktifkan situs:

```bash
sudo nano /etc/nginx/nginx.conf
```

Cari dan hapus tanda `#` pada baris:

```nginx
include /etc/nginx/sites-enabled/*;
```

Simpan dan tutup file.

6. Tes dan restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

## Langkah 5: Instalasi via Web Browser (Final)

1. Buka browser dan kunjungi alamat yang ditentukan di `APP_URL` (contoh: `http://localhost` atau alamat IP server Anda).
2. Pada halaman "Snipe-IT Pre-Flight", pastikan semua item berwarna hijau, lalu klik **Next: Create Database Tables**.
3. Isi formulir untuk membuat akun admin pertama, lalu klik **Next: Save User**.

**Selamat!** Instalasi Snipe-IT telah selesai.

## Troubleshooting Umum

### 1. ERR_TOO_MANY_REDIRECTS

Penyebab: Salah konfigurasi `APP_URL` atau tabel database belum dibuat.

Solusi:
- Periksa `APP_URL` di `.env` (pastikan menggunakan `http://`).
- Jalankan perintah berikut di direktori `/var/www/snipe-it`:

```bash
sudo php artisan migrate --force
sudo php artisan config:clear
sudo php artisan cache:clear
```

- Hapus cache browser.

### 2. Halaman Default Apache Muncul

Penyebab: Apache2 berjalan dan menggunakan port 80.

Solusi:

```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo systemctl restart nginx
```

### 3. Error Access Denied untuk Database

Penyebab: Pengguna database tidak memiliki hak akses.

Solusi: Ulangi perintah di Langkah 2 untuk memberikan hak akses:

```sql
GRANT ALL PRIVILEGES ON snipeit_db.* TO 'snipeit_user'@'localhost';
FLUSH PRIVILEGES;
```

### 4. Mengakses dari PC Lain Mengarah ke localhost

Penyebab: Konfigurasi belum disesuaikan untuk akses jaringan.

Solusi:
- Edit file `.env`:

```bash
sudo nano /var/www/snipe-it/.env
```

Ubah `APP_URL=http://localhost` menjadi `APP_URL=http://ALAMAT_IP_SERVER_ANDA`.

- Edit file konfigurasi Nginx:

```bash
sudo nano /etc/nginx/sites-available/snipe-it
```

Ubah `server_name localhost;` menjadi `server_name ALAMAT_IP_SERVER_ANDA;`.

- Terapkan perubahan:

```bash
cd /var/www/snipe-it
sudo php artisan config:clear
sudo systemctl restart php8.2-fpm
sudo systemctl restart nginx
```
