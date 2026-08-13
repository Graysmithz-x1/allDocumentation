# 🛠️ Troubleshoot Lab Phishing: GoPhish + MailHog (Docker Windows)

Dokumen ini adalah catatan referensi teknis pribadi untuk memetakan error jaringan kontainer, kegagalan SMTP, dan resolusi masalah arsitektur jaringan terisolasi pada lingkungan Docker Desktop Windows.

---

##  Error 1: `dial tcp: lookup mailhog: no such host`

###  Analisis Masalah
Pesan kesalahan ini terjadi di dalam antarmuka GoPhish saat mencoba mengeksekusi *Send Test Email*. Kontainer GoPhish tidak mampu menerjemahkan atau mendeteksi keberadaan host bernama `mailhog`. 
Hal ini disebabkan oleh masalah **Isolasi Jaringan Kontainer (Container Network Isolation)**. GoPhish mencoba mencari `mailhog` di dalam DNS internalnya, namun kedua kontainer tersebut belum berada dalam satu ruang jaringan inter-kontainer (*Bridge Network*) yang terintegrasi atau dependensi *service* belum siap sepenuhnya.

###  Solusi Permanen
1. Pastikan kedua kontainer dideklarasikan di dalam file `docker-compose.yml` yang sama.
2. Pastikan kontainer GoPhish memiliki instruksi `depends_on` yang mengarah ke `mailhog` untuk menjamin MailHog menyala lebih dahulu dan terdaftar di DNS Docker internal.
3. Di dalam parameter **Host** pada *Sending Profile* GoPhish, gunakan penamaan *service* resmi dari Docker Compose:
   ```text
   mailhog:1025
   ```

---

##  Error 2: Format Email Rusak pada Kolom "Send Test Email to"

###  Analisis Masalah
Saat melakukan pengujian pengiriman email, sistem GoPhish menolak input dan memecah satu alamat email menjadi beberapa kotak kecil (misal: kotak `david`, kotak `ngadaiin`, kotak `david@megafish`). 
Ini terjadi karena adanya **Karakter Spasi atau Whitespace** tak sengaja saat mengetik alamat tujuan. GoPhish menggunakan spasi sebagai pembatas (*delimiter*) untuk pengiriman email massal, sehingga alamat tunggal yang memiliki spasi akan dianggap sebagai beberapa baris surel yang cacat.

###  Solusi Permanen
1. Bersihkan seluruh kotak kecil yang rusak pada formulir *pop-up*.
2. Ketik satu alamat email target secara utuh tanpa spasi tunggal pun di awal, tengah, atau akhir kalimat (Contoh: `david@megafish.com`).
3. Langsung tekan tombol **Send** tanpa menekan tombol *Spacebar* atau *Enter* setelah mengetik.

---

##  Error 3: `Invalid SMTP server address`

###  Analisis Masalah
GoPhish menolak mentah-mentah pengisian kolom Host dan memunculkan peringatan format alamat tidak valid. 
Hal ini dipicu oleh kesalahan **Sintaksis Penulisan Host**. GoPhish menggunakan sistem *string parsing* yang sangat ketat terhadap pemisah alamat IP dan Port. Adanya spasi di sekitar karakter titik dua (`:`) akan membuat fungsi pembaca kode internal GoPhish gagal mengenali parameter port.

###  Solusi Permanen
Pastikan tidak ada spasi di sekitar tanda titik dua. Tulis dengan format *raw text* yang solid:
*   *Salah:* `127.0.0.1 : 1025` atau `localhost : 1025`
*    *Benar:* `127.0.0.1:1025` atau `localhost:1025`

---

##  Error 4: `Max connection attempts exceeded - dial tcp 127.0.0.1:8025: connect: connection refused`

###  Analisis Masalah
Koneksi ditolak keras (*connection refused*) oleh sistem. Ada dua kesalahan arsitektur jaringan yang bertubrukan di sini:
1. **Kesalahan Alokasi Port:** Port `8025` adalah pintu masuk untuk **Web UI / Dashboard** MailHog (tempat kita melihat kotak masuk lewat browser). GoPhish tidak bisa mengirim protokol email SMTP ke port web. Port khusus lalu lintas SMTP MailHog adalah `1025`.
2. **Kesalahan Penafsiran Loopback Kontainer:** Memasukkan IP `127.0.0.1` ke dalam aplikasi GoPhish yang terbungkus Docker akan membuat kontainer tersebut mencari MailHog di dalam dirinya sendiri (*localhost kontainer GoPhish itu sendiri*), bukan ke Windows utama atau ke kontainer MailHog seberang. Karena GoPhish tidak menjalankan SMTP internal, koneksi dibatalkan secara otomatis oleh sistem.

###  Solusi Permanen
Gunakan jalur DNS internal Docker Bridge atau manfaatkan gerbang khusus Docker Windows untuk melompati batasan *loopback*:
*   **Opsi Jalur Kontainer (Sangat Direkomendasikan):** `mailhog:1025`
*   **Opsi Jalur Host Windows (Alternatif):** `host.docker.internal:1025`

---

##  Masalah 5: Tombol Email Tidak Berinteraksi / Link `?rid=` Terputus

###  Analisis Masalah
Email berhasil masuk ke MailHog, namun tombol utama (seperti *"Verify my email address"*) mati, tidak bisa diklik, atau hanya memunculkan kode parameter menggantung seperti `?rid=j8OIPY1` tanpa alamat situs pengarah.
Ini terjadi karena **Parameter URL Kampanye Kosong / Salah**. Saat merancang *New Campaign*, kolom **URL** dibiarkan kosong atau salah isi. GoPhish membutuhkan alamat IP/Domain *Landing Page* komputer penyerang pada kolom tersebut untuk disuntikkan ke dalam tag HTML `{{.URL}}` sebagai pengarah jalur bagi korban.

###  Solusi Permanen
Saat mengeksekusi pembuatan **New Campaign**, pastikan wajib mengisi kolom **URL** dengan alamat server web penyerang yang valid (sesuai port *landing page* pada docker-compose, yaitu port 80). 
Untuk simulasi lab lokal Docker Windows, isi dengan:
```text
http://127.0.0.1
```
*(Jangan gunakan HTTPS jika belum mengonfigurasi sertifikat SSL lokal pada kontainer penyerang).*

---
*Catatan: Simpan file ini di folder utama lab sebagai dokumentasi taktis investigasi insiden.*...
