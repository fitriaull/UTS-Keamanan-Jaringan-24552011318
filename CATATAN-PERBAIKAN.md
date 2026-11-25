# Catatan perbaikan terhadap dokumen soal asli

Dokumen soal sendiri menulis "Kode di dokumen ini belum diuji di Docker."
Saya tidak bisa menjalankan Docker langsung di lingkungan chat ini (daemon
Docker tidak aktif di sandbox), tapi dari membaca kodenya saya menemukan dua
hal yang kemungkinan akan gagal saat benar-benar dijalankan. Perbaikannya
sudah saya terapkan di file-file lab ini:

1. **`netstat` tidak ada di `nginx:alpine` polos.**
   Tugas 6 (B3, SYN flood) menjalankan `docker compose exec target netstat -ant`,
   tapi image `nginx:alpine` resmi tidak menyertakan `net-tools`. Perbaikan:
   dibuat image turunan `uts-target` (`nginx:alpine` + `net-tools`) khusus untuk
   `b/b3-syn-flood/docker-compose.yml`. `b1-compose` tetap pakai `nginx:alpine`
   polos karena di situ hanya dipakai `wget`, bukan `netstat`.

2. **`arp -a` butuh paket `net-tools` di `uts-kali-tools`.**
   Tugas 3 (ARP spoofing) memakai `arp -a` dan `ping`, tapi Dockerfile
   `uts-kali-tools` di dokumen asli tidak menyertakan `net-tools` atau
   `iputils-ping`. Perbaikan: ditambahkan ke daftar `apt-get install` di
   `images/uts-kali-tools/Dockerfile`.

3. **`docker inspect -f` dengan nama network berisi tanda hubung (`-`).**
   Perintah seperti `docker inspect -f '{{.NetworkSettings.Networks.lab-arp-k2.IPAddress}}'`
   akan gagal di-parse oleh Go template karena `lab-arp-k2` bukan identifier
   yang valid (ada tanda hubungnya). Bentuk yang benar memakai fungsi `index`:

   ```bash
   docker inspect -f '{{(index .NetworkSettings.Networks "lab-arp-k2").IPAddress}}' korban-k2
   ```

   Semua perintah di `PERINTAH-KASUS-2.md` sudah memakai bentuk yang benar ini.

Tidak ada perubahan pada logika soal atau rubrik — ini murni perbaikan teknis
supaya perintahnya benar-benar bisa dijalankan dan diulang oleh penguji, sesuai
yang diminta di bagian "Pengurangan nilai" dokumen soal.
