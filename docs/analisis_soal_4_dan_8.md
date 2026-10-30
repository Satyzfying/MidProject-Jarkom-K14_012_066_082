# 📡 Analisis Teknis Soal 4 & 8: DHCP Services dan Layanan Aplikasi Smart City (Kelompok 14)

Dokumen ini mencatat implementasi bagian **Orang 3 (NRP 082)**: DHCP Services (Soal 4) dan
Layanan Aplikasi Smart City (Soal 8). Seluruh konfigurasi diuji langsung pada topologi live
`K14_VLSM.pkt` di Cisco Packet Tracer 9.0.1.

---

## 1. Soal 4 — DHCP Services

### A. Prinsip yang Diterapkan
| Ketentuan Soal | Implementasi |
|:---|:---|
| Interface router dan seluruh server memakai **IP statis** | Seluruh router dan 6 server tetap statis; tidak ada pool yang mencakup alamat server (`192.112.239.2-4`, `192.112.241.2-4`) |
| Perangkat pengguna mendapat **IP dinamis** | Seluruh **36 PC** di 12 LAN pengguna diubah dari statis ke DHCP (`mode dhcp`) |
| Lengkap dengan **gateway dan DNS** | Setiap pool mengirim `default-router` LAN masing-masing dan `dns-server 192.112.239.2` |
| **Tanpa bentrok** dengan IP statis | Blok `.1 - .20` di setiap LAN di-`exclude` (gateway + perangkat statis) |
| **CCTV Kota** dilayani DHCP server di LAN Server & Storage DC, **bukan router** | R-DC hanya menjadi **relay** (`ip helper-address 192.112.239.2`); pool CCTV berada di Server dan Storage DC 1 |

### B. Tabel Pool DHCP per Router

| Router | Nama Pool | Network / Prefix | Gateway | DNS | Excluded |
|:---|:---|:---|:---:|:---:|:---|
| `R-PUSAT` | `PEGAWAI-BALAIKOTA` | `192.112.224.0/21` | `192.112.224.1` | `192.112.239.2` | `.1 - .20` |
| `R-UTARA` | `WIFI-UTARA` | `192.112.0.0/18` | `192.112.0.1` | `192.112.239.2` | `.1 - .20` |
| `R-UTARA` | `SENSOR-LALIN` | `192.112.236.0/23` | `192.112.236.1` | `192.112.239.2` | `.1 - .20` |
| `R-SELATAN` | `WIFI-SELATAN` | `192.112.64.0/19` | `192.112.64.1` | `192.112.239.2` | `.1 - .20` |
| `R-SELATAN` | `PJU-SELATAN` | `192.112.240.128/25` | `192.112.240.129` | `192.112.239.2` | `.129 - .140` |
| `R-TIMUR` | `WIFI-TIMUR` | `192.112.128.0/19` | `192.112.128.1` | `192.112.239.2` | `.1 - .20` |
| `R-TIMUR` | `SENSOR-BANJIR` | `192.112.232.0/22` | `192.112.232.1` | `192.112.239.2` | `.1 - .20` |
| `R-BARAT` | `WIFI-BARAT` | `192.112.192.0/19` | `192.112.192.1` | `192.112.239.2` | `.1 - .20` |
| `R-BARAT` | `PUSKESMAS-BARAT` | `192.112.238.0/24` | `192.112.238.1` | `192.112.239.2` | `.1 - .20` |
| `R-PELABUHAN` | `SCADA-PELABUHAN` | `192.112.160.0/19` | `192.112.160.1` | `192.112.239.2` | `.1 - .20` |
| `R-PELABUHAN` | `KANTOR-PELABUHAN` | `192.112.240.0/25` | `192.112.240.1` | `192.112.239.2` | `.1 - .20` |
| `R-DC` | — *(relay)* | `—` | `—` | `—` | `ip helper-address 192.112.239.2` |
| `Server dan Storage DC 1` | `CCTV-KOTA` | `192.112.96.0/19` | `192.112.96.1` | `192.112.239.2` | start `.100`, 6000 IP (`.100 - .119.211`) |

> Konfigurasi lengkap: [`nodes/dhcp_all_routers.cfg`](../nodes/dhcp_all_routers.cfg)

### C. Mengapa CCTV Memakai Relay, Bukan Pool Router?
Soal secara eksplisit meminta subnet CCTV Kota dilayani **server DHCP di LAN Server & Storage DC**.
Karena R-DC berada di antara LAN CCTV dan LAN server, R-DC dikonfigurasi sebagai **DHCP Relay Agent**
(`ip helper-address 192.112.239.2`) pada `GigabitEthernet0/1`. Mekanismenya:

1. PC CCTV mengirim **DHCP Discover** broadcast (`255.255.255.255`) di LAN CCTV.
2. R-DC menangkap broadcast tersebut dan **mengubahnya menjadi unicast** ke `192.112.239.2`.
3. Server DC 1 menjawab dengan **DHCP Offer/ACK** berisi IP, gateway, dan DNS.
4. R-DC meneruskan jawaban ke klien; klien menyelesaikan proses DHCP.

Router tidak membuat pool apa pun untuk CCTV (`show ip dhcp pool` pada R-DC kosong), sehingga
ketentuan "bukan dari router" terpenuhi secara literal.

### D. Bukti Pengujian DHCP

**1. Lease pada DHCP Server DC (CCTV melalui relay):**
```text
Server dan Storage DC 1 — pool CCTV-KOTA
192.112.96.100  MAC 00E0.B0AB.9413   (CCTV Kota 1)
192.112.96.101  MAC 0005.5EB1.14A0   (CCTV Kota 2)
192.112.96.102  MAC 0005.5EC8.A0CA   (CCTV Kota 3)
3 lease(s)
```

**2. Lease pada router (contoh R-UTARA):**
```text
WIFI-UTARA     192.112.0.22 / .24 / .25
SENSOR-LALIN   192.112.236.21 / .22 / .23
6 lease(s)
```

**3. `ipconfig /all` pada klien (WiFi Publik Selatan 1):**
```text
IPv4 Address........: 192.112.64.21
Subnet Mask.........: 255.255.224.0
Default Gateway.....: 192.112.64.1
DHCP Server.........: 192.112.64.1
DNS Servers.........: 192.112.239.2
```
Bukti bahwa klien menerima **IP + gateway + DNS** secara otomatis.

---

## 2. Soal 8 — Layanan Aplikasi Smart City

### A. Domain dan Pemetaan Server

Domain internal kelompok: **`14.smartcity.lab`** (ditambah alias `surabaya.go.id` agar portal
dapat dibuka dengan nama domain kota).

| Server | IP Statis | Peran Layanan |
|:---|:---:|:---|
| `Server dan Storage DC 1` | `192.112.239.2` | **DHCP** (CCTV) + **DNS** |
| `Server dan Storage DC 2` | `192.112.239.3` | **FTP** (penyimpanan rekaman CCTV) |
| `Server dan Storage DC 3` | `192.112.239.4` | **Mail** (SMTP + POP3) |
| `Server Balai Kota 1` | `192.112.241.2` | **Web** — Portal Smart City Surabaya |
| `Server Balai Kota 2` | `192.112.241.3` | Cadangan |
| `Server Balai Kota 3` | `192.112.241.4` | Cadangan |

### B. DNS (Server dan Storage DC 1)
Record `A` yang terdaftar dan terbaca tersimpan (`stored: true`):

| Nama | Tipe | Value |
|:---|:---:|:---|
| `www.14.smartcity.lab` | A | `192.112.241.2` |
| `ftp.14.smartcity.lab` | A | `192.112.239.3` |
| `mail.14.smartcity.lab` | A | `192.112.239.4` |
| `14.smartcity.lab` | A | `192.112.241.2` |
| `www.surabaya.go.id` | A | `192.112.241.2` |
| `ftp.surabaya.go.id` | A | `192.112.239.3` |
| `mail.surabaya.go.id` | A | `192.112.239.4` |

**Bukti resolusi dari wilayah lain:**
```text
C:\> nslookup www.14.smartcity.lab
Server:  [192.112.239.2]
Name:    www.14.smartcity.lab
Address: 192.112.241.2

C:\> nslookup mail.14.smartcity.lab
Server:  [192.112.239.2]
Name:    mail.14.smartcity.lab
Address: 192.112.239.4
```

### C. Web Server (Server Balai Kota 1 — `192.112.241.2`)
Halaman `index.html` diganti dengan **"Portal Smart City Surabaya"** yang memuat **nomor kelompok 14**
dan daftar NRP anggota. HTTP aktif pada port 80.

**Bukti akses lintas wilayah:**

| Klien | URL | Hasil |
|:---|:---|:---|
| `WiFi Publik Selatan 1` | `http://www.14.smartcity.lab` | ✅ halaman termuat |
| `WiFi Publik Utara 1` | `http://www.surabaya.go.id` | ✅ halaman termuat |
| `Sensor dan SCADA Pelabuhan 1` | `http://www.14.smartcity.lab` | ✅ halaman termuat (via GRE Tunnel) |

### D. FTP (Server dan Storage DC 2 — `192.112.239.3`)
| Username | Password | Permission |
|:---|:---|:---|
| `cctv` | `K14cctv` | `RWDNL` (Read, Write, Delete, reName, List) |
| `pegawai` | `K14pegawai` | `RWDNL` |

Permission **W (Write)** inilah yang memungkinkan perangkat LAN CCTV Kota mengunggah rekaman.
**Bukti login dan listing dari CCTV Kota 1 (menggunakan nama domain):**
```text
C:\> ftp ftp.14.smartcity.lab
220- Welcome to PT Ftp server
Username: cctv
Password: *******
230- Logged in
ftp> dir
Listing /ftp directory from ftp.14.smartcity.lab: ... (35 berkas)
```

> **Catatan simulator:** PC-PT di Packet Tracer tidak memiliki berkas lokal di `C:\`, sehingga
> perintah `put` menghasilkan `Error opening ... (No such file or directory)`. Ini keterbatasan
> simulator, bukan server: autentikasi dan hak tulis sudah terbukti diterima server. Untuk demo
> unggah berkas nyata, gunakan **Text Editor** pada PC di GUI PT untuk membuat berkas terlebih
> dahulu, lalu `put <nama-berkas>`.

### E. Mail (Server dan Storage DC 3 — `192.112.239.4`)
- **Domain email:** `14.smartcity.lab`
- **SMTP:** On, **POP3:** On
- **Akun:** `balai1`, `balai2`, `balai3`, `pelabuhan1`, `pelabuhan2`, `pelabuhan3` (password `K14mail`)
- **Klien email terkonfigurasi:** seluruh 3 PC di LAN Pegawai Balai Kota dan 3 PC di LAN Kantor Pelabuhan.

**Bukti pengiriman dua arah:**

| Arah | Pengirim | Penerima | Hasil |
|:---|:---|:---|:---|
| Balai Kota ➔ Pelabuhan | `balai1@14.smartcity.lab` (`Pegawai Balai Kota 1`) | `pelabuhan1@14.smartcity.lab` | ✅ `sent to pelabuhan1@...`, POP3 fetch OK |
| Pelabuhan ➔ Balai Kota | `pelabuhan1@14.smartcity.lab` (`Kantor Pelabuhan 1`) | `balai1@14.smartcity.lab` | ✅ `sent to balai1@...`, POP3 fetch OK |
| Balai Kota ➔ Pelabuhan | `balai3@14.smartcity.lab` (`Pegawai Balai Kota 3`) | `pelabuhan3@14.smartcity.lab` | ✅ `sent to pelabuhan3@...`, POP3 fetch OK |

Server SMTP/POP3 diakses dengan nama domain `mail.14.smartcity.lab` (bukan IP), sehingga
sekaligus membuktikan jalur DNS bekerja pada klien email.

---

## 3. Ringkasan Hasil Uji

| Butir Soal | Bukti | Status |
|:---|:---|:---:|
| DHCP pool seluruh LAN pengguna | lease terbaca di R-PUSAT, R-UTARA, R-SELATAN, R-BARAT, R-PELABUHAN, R-TIMUR | ✅ |
| Gateway + DNS otomatis | `ipconfig /all` klien | ✅ |
| CCTV via DHCP server DC | 3 lease di pool `CCTV-KOTA`, R-DC tanpa pool | ✅ |
| Tanpa bentrok IP statis | exclude `.1-.20`, server di luar pool | ✅ |
| DNS `www`/`ftp`/`mail` dari semua wilayah | `nslookup` dari Selatan & Pelabuhan | ✅ |
| Web Portal K14 by domain | browser dari Selatan, Utara, Pelabuhan | ✅ |
| FTP login + hak tulis dari LAN CCTV | `ftp ftp.14.smartcity.lab` dari CCTV Kota 1 | ✅ |
| Email Balai Kota ↔ Pelabuhan | kirim + POP3 dua arah | ✅ |
| Akses internet setelah DHCP | `ping 8.8.8.8` dari CCTV Kota 1, Pegawai Balai Kota 1, Server Balai Kota 1 | ✅ |

**Bukti visual topologi:** [`images/soal-8/topologi-k14-vlsm.png`](../images/soal-8/topologi-k14-vlsm.png)
(diambil langsung dari canvas Packet Tracer, termasuk link L4 menuju R-TIMUR).

---

## 4. Temuan di Luar Lingkup

### A. NAT Dua SVI Balai Kota (sudah diperbaiki)
Saat verifikasi, PC di LAN **Pegawai Balai Kota** (`Vlan21`) dan **Server Balai Kota** (`Vlan27`)
belum bisa mencapai `8.8.8.8`, sedangkan LAN CCTV bisa. Penyebabnya:

```text
R-PUSAT# show ip nat statistics
Inside Interfaces: GigabitEthernet0/0/0, GigabitEthernet0/0/1, Vlan30, Vlan50
```

`Vlan21` dan `Vlan27` belum ditandai `ip nat inside`, sehingga paket dari kedua LAN keluar ke ISP
tanpa diterjemahkan dan tidak punya jalur balik. Perbaikan dua baris:

```cisco
interface Vlan21
 ip nat inside
interface Vlan27
 ip nat inside
```

Hasil: `ping 8.8.8.8` dari `Pegawai Balai Kota 1` dan `Server Balai Kota 1` kini **100% sukses**.
Tidak ada perubahan pada Tunnel0, ISAKMP, crypto map, atau routing.

### B. Link Backbone L4 (`R-PUSAT` ↔ `R-TIMUR`) — diperbaiki

**Kondisi awal:** `R-TIMUR` hanya terhubung ke `SW-WiFi-Timur` dan `SW-Banjir`; `GigabitEthernet0/0`-nya
tidak terkabel, sementara seluruh switchport `R-PUSAT` (`Gi0/1/0`–`Gi0/1/3`) sudah terpakai.
Wilayah **Timur (WiFi Publik Timur + Sensor Banjir Timur)** terisolasi:

```text
C:\> ping 8.8.8.8        (dari WiFi Publik Timur 1)
Reply from 192.112.128.1: Destination host unreachable.
```

PC di Timur tetap mendapat lease DHCP lokal dari R-TIMUR (pool berjalan), tetapi tidak dapat mencapai
DNS, web, FTP, mail, maupun internet.

**Perbaikan yang diterapkan (VLAN 40 + trunk lewat `SW-Pegawai`):**

```cisco
! SW-Pegawai
vlan 21
 name PEGAWAI
vlan 40
 name L4-TIMUR
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 21
 switchport trunk allowed vlan 21,40
interface range FastEthernet0/1 - 3
 switchport access vlan 21
interface FastEthernet0/10
 switchport access vlan 40

! R-PUSAT
interface GigabitEthernet0/1/2
 switchport mode trunk
 switchport trunk native vlan 21
 switchport trunk allowed vlan 21,40
interface Vlan40
 ip address 192.112.241.45 255.255.255.252
 ip nat inside
 no shutdown

! Kabel
R-TIMUR Gi0/0 <--straight--> SW-Pegawai Fa0/10
```

`R-TIMUR Gi0/0` sebenarnya **sudah** dikonfigurasi `192.112.241.46/30`, hanya belum punya kabel.
Setelah kabel terpasang, adjacency OSPF langsung terbentuk (tanpa perubahan DHCP/aplikasi):

```text
R-TIMUR# show ip ospf neighbor
Neighbor ID     Pri   State      Dead Time   Address         Interface
203.0.113.57      1   FULL/DR    00:00:32    192.112.241.45  GigabitEthernet0/0

R-TIMUR# show ip route
O*E2 0.0.0.0/0 [110/1] via 192.112.241.45, ... GigabitEthernet0/0
```

Hasil akhir: `ping 8.8.8.8`, resolusi DNS, dan akses `http://www.14.smartcity.lab` dari
`WiFi Publik Timur 1` berhasil. Sambungan L2 ini tetap membentuk **L4 logis point-to-point /30**
sesuai spreadsheet (`192.112.241.44/30`), hanya jalur fisiknya melewati `SW-Pegawai` karena
`R-PUSAT` tidak memiliki port bebas.
