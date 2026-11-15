# Cheat Sheet Uji 2 Individu (Materi Orang 3: DHCP & Layanan Aplikasi)

**Waktu Ujian: 10 Menit | Bobot: 15 Poin | Target: Temukan, Perbaiki, & Jelaskan**

---

## Perintah Sakti Pertama

```cisco
enable
show ip dhcp binding
show ip dhcp pool
show running-config | section dhcp
```

Di PC: `ipconfig /all`, `ipconfig /renew`, `nslookup`, `ping`.

---

## Skenario 1: PC Tidak Mendapat IP DHCP

### Gejala
`ipconfig /all` menampilkan `IPv4 Address: 0.0.0.0` atau `169.254.x.x`.

### Lokasi Pengecekan
- Di router LAN: `show ip dhcp binding`, `show ip dhcp pool`.
- Di PC: `ipconfig /renew`.

### Sabotase yang Sering Dibuat Aslab

| Sabotase | Cara Mengetahui | Perbaikan Cepat |
|---|---|---|
| DHCP service mati | `show ip dhcp pool` kosong / tidak ada binding | `service dhcp` |
| Pool salah network/gateway | `show ip dhcp pool` tidak sesuai VLSM | Perbaiki `network`, `default-router`, `dns-server` |
| Excluded address salah | Pool kehabisan IP | `ip dhcp excluded-address <awal> <akhir>` |
| Relay CCTV hilang | Klien CCTV tidak dapat IP dari server DC | Di interface CCTV: `ip helper-address 192.112.239.2` |
| Server DHCP DC mati | Pool `CCTV-KOTA` tidak ada | Aktifkan DHCP server di `Server dan Storage DC 1` |

---

## Skenario 2: DNS Tidak Menjawab

### Gejala
`nslookup www.14.smartcity.lab` timeout / `Server unknown`.

### Lokasi Pengecekan
- Di PC: `nslookup www.14.smartcity.lab`.
- Di server DNS: pastikan service On dan record A ada.

### Sabotase yang Sering Dibuat Aslab

| Sabotase | Cara Mengetahui | Perbaikan Cepat |
|---|---|---|
| DNS server off | `nslookup` timeout | Aktifkan DNS di `Server dan Storage DC 1` |
| Record A hilang | Nama tidak resolve | Tambahkan `www`, `ftp`, `mail`, dan apex untuk kedua domain |
| DNS client salah | `ipconfig /all` menunjuk DNS salah | Perbaiki `dns-server 192.112.239.2` di pool DHCP |
| Domain salah ketik | `nslookup` nama lain | Gunakan `14.smartcity.lab` atau `surabaya.go.id` |

---

## Skenario 3: Web / FTP / Mail Gagal

### Gejala
Domain resolve, tetapi halaman web tidak terbuka atau login FTP/mail gagal.

### Sabotase yang Sering Dibuat Aslab

| Layanan | Sabotase | Perbaikan Cepat |
|---|---|---|
| Web | HTTP service off | Aktifkan HTTP/HTTPS di `Server Balai Kota 1` |
| Web | `index.html` kosong/salah | Kembalikan halaman "Portal Smart City Surabaya" + identitas anggota |
| FTP | Service off / user salah | Aktifkan FTP, cek user `cctv` / `pegawai` dan permission `RWDNL` |
| Mail | SMTP/POP3 off | Aktifkan SMTP/POP3, cek domain `14.smartcity.lab` dan akun `balai1`-`balai3`, `pelabuhan1`-`pelabuhan3` |
| Semua | Rute/DNS putus | Cek `ping` ke server dan `nslookup` |

---

## Cara Menjelaskan ke Dosen (30 Detik)

> "Izin menjelaskan, Pak/Bu. Klien [nama PC] tidak mendapat layanan [DHCP/DNS/Web/FTP/Mail] karena [pool/relay/record/service] di [nama server/router] [mati/salah]. Saya perbaiki dengan [perintah], lalu saya verifikasi lewat `ipconfig /all`, `nslookup`, dan akses layanan; hasilnya sudah normal."
