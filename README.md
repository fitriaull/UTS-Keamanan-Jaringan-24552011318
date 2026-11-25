# uts-network-lab

Lab Docker untuk UTS Praktik Infrastruktur & Network Attacks. Jalankan **hanya**
di lab lokal (laptop/VM Anda sendiri) — jangan pindai atau serang host di luar lab.

> Catatan: paket ini dibangun ulang dari dokumen soal, dengan dua perbaikan kecil
> yang disebutkan di `CATATAN-PERBAIKAN.md` supaya benar-benar bisa jalan di Docker.

## 1. Build image (sekali di awal)

```bash
cd uts-network-lab
docker build -t uts-openssl images/uts-openssl
docker build -t uts-kali-tools images/uts-kali-tools
docker build -t uts-dnsmasq images/uts-dnsmasq
docker pull alpine
docker pull nginx:alpine
```

Jaringan Bagian B memakai `internal: true` (tanpa akses internet), jadi semua
paket harus sudah ada di image sebelum dipakai — itulah sebabnya build dilakukan
di awal, bukan saat container sudah berjalan di jaringan internal.

## 2. Struktur folder

```
uts-network-lab/
├── images/
│   ├── uts-openssl/      (Pertemuan 2: OpenSSL)
│   ├── uts-kali-tools/   (Pertemuan 3-7: hping3, dig, tcpdump, scapy, aircrack-ng, arpspoof)
│   ├── uts-dnsmasq/      (Pertemuan 7: resolver DNS lab)
│   └── uts-target/       (perbaikan: nginx:alpine + net-tools, dipakai di B3/Tugas 6)
├── b/
│   ├── b1-compose/       (Tugas 1 / Pertemuan 1)
│   ├── b3-syn-flood/     (Tugas 6 / Pertemuan 6)
│   └── b4-dns-exfil/     (Tugas 7 / Pertemuan 7)
└── kasus/
    ├── k5-scapy/         (Tugas 5 / Pertemuan 5)
    └── k9-dns-detect/    (dipakai di Tugas 7 / Pertemuan 7)
```

## 3. Kasus Anda

NIM **24552011318** → jumlah digit 32 → sisa bagi 10 = **2** → **Kasus 2 (k2)**.

Parameter: container `target-k2`, volume `vol-k2`, file `laporan-k2.txt`,
kata sandi `kunci-k2-2026`, domain `k2.exfil.lab`.

Ikuti perintah lengkap per tugas di `PERINTAH-KASUS-2.md`, lalu isi `jawaban.md`
dengan output asli dari terminal Anda.
