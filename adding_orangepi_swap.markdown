# Panduan Menambahkan Swap di Orange Pi 3 LTS

Dokumen ini menjelaskan cara membuat dan mengaktifkan file swap (ruang disk yang digunakan sebagai RAM virtual) di Orange Pi 3 LTS yang menjalankan sistem operasi berbasis Linux seperti Armbian atau Ubuntu. Swap sangat berguna untuk mencegah sistem melambat atau crash ketika RAM fisik penuh.

## Langkah 1: Cek Status Swap Saat Ini
Sebelum membuat swap baru, penting untuk memeriksa apakah sistem sudah memiliki swap yang aktif.

Buka terminal dan jalankan salah satu perintah berikut:

```bash
sudo swapon --show
```

Atau, untuk melihat penggunaan memori dan swap secara keseluruhan:

```bash
free -h
```

Jika kedua perintah tersebut tidak menghasilkan output apa pun atau menunjukkan nilai 0 pada baris Swap, berarti sistem Anda belum memiliki ruang swap aktif.

## Langkah 2: Buat File untuk Swap
Kita akan membuat sebuah file yang akan dialokasikan sebagai ruang swap. Ukuran file ini tergantung pada kebutuhan Anda. Aturan umum yang baik adalah 1x hingga 2x dari ukuran RAM fisik. Misalnya, jika Orange Pi Anda memiliki RAM 2GB, membuat swap 2GB adalah pilihan yang aman.

Gunakan perintah `fallocate` untuk membuat file bernama `swapfile` di direktori root (`/`) dengan ukuran 2GB.

```bash
sudo fallocate -l 2G /swapfile
```

**Catatan**: Jika perintah `fallocate` gagal, Anda bisa menggunakan alternatif `dd` yang sedikit lebih lambat:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1024 count=2097152
```

## Langkah 3: Atur Izin File Swap
Untuk alasan keamanan, file swap tidak boleh bisa dibaca oleh pengguna selain root. Kita perlu mengunci izin file tersebut.

```bash
sudo chmod 600 /swapfile
```

Perintah ini memastikan hanya user root yang memiliki hak baca dan tulis terhadap file `swapfile`.

## Langkah 4: Format File Menjadi Swap
Sekarang, kita perlu memberitahu sistem bahwa file ini akan digunakan sebagai ruang swap.

```bash
sudo mkswap /swapfile
```

Anda akan melihat output yang mengonfirmasi bahwa ruang swap telah berhasil dibuat.

## Langkah 5: Aktifkan Swap
Setelah file diformat, aktifkan agar sistem mulai menggunakannya.

```bash
sudo swapon /swapfile
```

## Langkah 6: Verifikasi Swap Sudah Aktif
Jalankan kembali perintah dari Langkah 1 untuk memastikan swap sudah aktif dan dikenali oleh sistem.

```bash
sudo swapon --show
```

**Output yang diharapkan**:

```
NAME       TYPE SIZE USED PRIO
/swapfile file   2G   0B   -2
```

Dan juga:

```bash
free -h
```

**Output yang diharapkan** akan menunjukkan baris Swap dengan total ukuran 2G (atau ukuran yang Anda tentukan).

## Langkah 7: Jadikan Swap Permanen (Penting!)
Swap yang kita aktifkan tadi hanya bersifat sementara dan akan hilang setelah reboot. Untuk membuatnya permanen, kita perlu menambahkan entri ke file `/etc/fstab`.

Cadangkan file `fstab` terlebih dahulu (tindakan pencegahan):

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

Tambahkan konfigurasi swap ke `/etc/fstab`:

Gunakan perintah berikut untuk menambahkan baris yang diperlukan ke akhir file secara otomatis.

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Sekarang, ruang swap akan otomatis aktif setiap kali Orange Pi Anda dinyalakan.

## (Opsional) Mengatur "Swappiness"
Swappiness adalah parameter kernel yang mengatur seberapa agresif sistem menggunakan ruang swap. Nilainya berkisar dari 0 hingga 100.

- **100**: Sangat agresif, sering menggunakan swap.
- **0**: Hanya menggunakan swap jika benar-benar terpaksa (RAM hampir habis).

Untuk perangkat seperti Orange Pi dengan penyimpanan kartu SD yang lebih lambat dari RAM, disarankan untuk mengatur swappiness ke nilai yang lebih rendah (misalnya 10) agar sistem lebih memprioritaskan penggunaan RAM.

Cek nilai swappiness saat ini:

```bash
cat /proc/sys/vm/swappiness
```

(Defaultnya biasanya 60).

Ubah nilai swappiness menjadi 10 (sementara):

```bash
sudo sysctl vm.swappiness=10
```

Jadikan perubahan swappiness permanen:

Edit file `sysctl.conf`.

```bash
sudo nano /etc/sysctl.conf
```

Tambahkan baris berikut di akhir file:

```
vm.swappiness=10
```

Simpan file (Ctrl+X, lalu Y, lalu Enter) dan reboot atau jalankan `sudo sysctl -p` untuk menerapkan perubahan.