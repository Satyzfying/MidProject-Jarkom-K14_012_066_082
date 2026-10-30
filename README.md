# MID PROJECT KOMUNIKASI DATA & JARINGAN KOMPUTER K14-012-066-082
### Implementasi Jaringan Smart City Pemerintah Kota Surabaya
**Semester Gasal 2026/2027**

---

## 👥 Anggota Kelompok 14

| No | Nama | NRP | Peran / Fokus Pengerjaan |
|:--:|:---|:---:|:---|
| 1 | Anggota 1 | `...012` | Infrastructure, Topologi WAN, Static Routing & NAT |
| 2 | Anggota 2 | `...066` | Subnetting Spreadsheet, GRE Tunnel, IPsec, & Dynamic OSPF |
| 3 | Anggota 3 | `...082` | DHCP Services, Smart City Applications (DNS, Web, FTP, Mail) & Documentation |

---

## 📂 Struktur Direktori Repositori

```text
.
├── docs/                               <-- Dokumen analisis teknis & panduan
│   ├── analisis_soal_6_tunnel.md       <-- Analisis GRE Tunnel & IPsec
│   ├── analisis_soal_7_ospf.md         <-- Analisis OSPF Area 0 & Failover L7
│   ├── analisis_soal_4_dan_8.md        <-- Analisis DHCP Services & Layanan Aplikasi
│   └── cheat_sheet_uji2_orang2.md      <-- Panduan Uji 2 Individu (10 Menit)
├── images/                             <-- Bukti screenshot pengujian mandiri
│   ├── soal-6/                         <-- Screenshot pengujian GRE & IPsec
│   ├── soal-7/                         <-- Screenshot pengujian OSPF & Failover
│   └── soal-8/                         <-- Screenshot DHCP & layanan aplikasi
├── nodes/                              <-- Konfigurasi Cisco IOS per perangkat (.cfg)
│   ├── R-PUSAT/
│   │   └── gre_tunnel.cfg
│   ├── R-PELABUHAN/
│   │   └── gre_tunnel.cfg
│   ├── ospf_all_routers.cfg            <-- Script OSPF lengkap 7 router
│   └── dhcp_all_routers.cfg            <-- Script DHCP 7 router + relay CCTV
├── result/                             <-- Berkas output utama yang dinilai
│   ├── K14_VLSM.pkt                    <-- File 1: Topologi Utama (Static + GRE + Apps)
│   ├── K14_VLSM_OSPF.pkt               <-- File 2: Varian Dynamic Routing OSPF + Link L7
│   └── UTS-KOMDAT-JARKOM-K14.xlsx      <-- Spreadsheet Subnetting Resmi (4 Sheet)
├── .gitignore
└── README.md
```

---

## 1. Topologi & Skema Pengalamatan

### A. Pertimbangan Pemilihan Skema: **VLSM (Variable Length Subnet Mask)**
Berdasarkan hasil analisis kebutuhan riil Smart City Surabaya:
- **Kebutuhan Aktual**: Seluruh kota membutuhkan **61.752 alamat IP host**.
- **Kapasitas Alokasi VLSM**:
  - VLSM mampu mengakomodasi seluruh 14 subnet LAN dan 7 link WAN hanya dalam **1 Blok Induk `/16` (`192.112.0.0/16`)** yang berkapasitas 65.536 alamat.
  - **Efisiensi Penggunaan**: **94,23%** (61.752 / 65.536), dengan sisa cadangan 3.784 alamat IP untuk ekspansi masa depan.
- **Perbandingan dengan Skema CIDR**:
  - Skema CIDR membagi blok per wilayah router secara kaku sehingga membutuhkan total alokasi **118.784 alamat**.
  - Hal ini memaksa pemakaian blok induk supernet `/15` (`192.112.0.0/15` berkapasitas 131.072 alamat), yang mengakibatkan pemborosan ruang alamat lebih dari **52%**.

> **Kesimpulan**: Skema **VLSM dipilih** sebagai implementasi final karena jauh lebih hemat, efisien, dan presisi dalam pemanfaatan ruang alamat IP publik/privat kota.

---

## 2. Desain Topologi Jaringan

Topologi mengadopsi arsitektur *Hub-and-Spoke* dengan **`R-PUSAT`** (Balai Kota) sebagai *Core Hub*:
- `R-PUSAT` terhubung langsung ke 5 router cabang via link GigabitEthernet (`L1` s.d. `L5`).
- `R-PUSAT` terhubung ke `R-ISP` via Uplink GigabitEthernet untuk akses simulasi internet (`8.8.8.8`).
- `R-PELABUHAN` terhubung ke `R-UTARA` via Link `L6`.
- Satu jalur virtual **GRE Tunnel** dibangun langsung antara `R-PUSAT` dan `R-PELABUHAN`.
- Pada varian OSPF, satu link cadangan **`L7`** menghubungkan `R-DC` dan `R-UTARA` untuk redundansi.

### 2.1 Daftar Alokasi Subnet LAN Kota (14 Subnet)

| No | Router | Wilayah / Layanan | Kebutuhan Host | Network ID & Prefix | Subnet Mask | Gateway | Rentang IP Host | Broadcast |
|:--:|:---|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | `R-UTARA` | WiFi Publik Utara | 11.000 | `192.112.0.0/18` | `255.255.192.0` | `192.112.0.1` | `.0.2` – `.63.254` | `192.112.63.255` |
| 2 | `R-SELATAN` | WiFi Publik Selatan | 6.500 | `192.112.64.0/19` | `255.255.224.0` | `192.112.64.1` | `.64.2` – `.95.254` | `192.112.95.255` |
| 3 | `R-DC` | CCTV Kota | 6.000 | `192.112.96.0/19` | `255.255.224.0` | `192.112.96.1` | `.96.2` – `.127.254` | `192.112.127.255` |
| 4 | `R-TIMUR` | WiFi Publik Timur | 5.500 | `192.112.128.0/19` | `255.255.224.0` | `192.112.128.1` | `.128.2` – `.159.254` | `192.112.159.255` |
| 5 | `R-PELABUHAN` | Sensor & SCADA Pelabuhan | 5.000 | `192.112.160.0/19` | `255.255.224.0` | `192.112.160.1` | `.160.2` – `.191.254` | `192.112.191.255` |
| 6 | `R-BARAT` | WiFi Publik Barat | 4.500 | `192.112.192.0/19` | `255.255.224.0` | `192.112.192.1` | `.192.2` – `.223.254` | `192.112.223.255` |
| 7 | `R-PUSAT` | Pegawai Balai Kota | 1.800 | `192.112.224.0/21` | `255.255.248.0` | `192.112.224.1` | `.224.2` – `.231.254` | `192.112.231.255` |
| 8 | `R-TIMUR` | Sensor Banjir Timur | 700 | `192.112.232.0/22` | `255.255.252.0` | `192.112.232.1` | `.232.2` – `.235.254` | `192.112.235.255` |
| 9 | `R-UTARA` | Sensor Lalu Lintas | 450 | `192.112.236.0/23` | `255.255.254.0` | `192.112.236.1` | `.236.2` – `.237.254` | `192.112.237.255` |
| 10 | `R-BARAT` | Puskesmas & Sekolah | 220 | `192.112.238.0/24` | `255.255.255.0` | `192.112.238.1` | `.238.2` – `.238.254` | `192.112.238.255` |
| 11 | `R-DC` | Server & Storage DC | 180 | `192.112.239.0/24` | `255.255.255.0` | `192.112.239.1` | `.239.2` – `.239.254` | `192.112.239.255` |
| 12 | `R-PELABUHAN` | Kantor Pelabuhan | 120 | `192.112.240.0/25` | `255.255.255.128` | `192.112.240.1` | `.240.2` – `.240.126` | `192.112.240.127` |
| 13 | `R-SELATAN` | PJU Pintar Selatan | 90 | `192.112.240.128/25` | `255.255.255.128` | `192.112.240.129` | `.240.130` – `.240.254` | `192.112.240.255` |
| 14 | `R-PUSAT` | Server Balai Kota | 20 | `192.112.241.0/27` | `255.255.255.224` | `192.112.241.1` | `.241.2` – `.241.30` | `192.112.241.31` |

### 2.2 Daftar Interkoneksi Link Point-to-Point

| Link | Segmen Koneksi | Subnet ID | IP Router Sisi A | IP Router Sisi B | Keterangan |
|:---:|:---|:---:|:---:|:---:|:---|
| **`L1`** | `R-PUSAT` <---> `R-DC` | `192.112.241.32/30` | `192.112.241.33` | `192.112.241.34` | Backbone Data Center |
| **`L2`** | `R-PUSAT` <---> `R-UTARA` | `192.112.241.36/30` | `192.112.241.37` | `192.112.241.38` | Backbone Wilayah Utara |
| **`L3`** | `R-PUSAT` <---> `R-SELATAN` | `192.112.241.40/30` | `192.112.241.41` | `192.112.241.42` | Backbone Wilayah Selatan |
| **`L4`** | `R-PUSAT` <---> `R-TIMUR` | `192.112.241.44/30` | `192.112.241.45` | `192.112.241.46` | Backbone Wilayah Timur |
| **`L5`** | `R-PUSAT` <---> `R-BARAT` | `192.112.241.48/30` | `192.112.241.49` | `192.112.241.50` | Backbone Wilayah Barat |
| **`L6`** | `R-UTARA` <---> `R-PELABUHAN` | `192.112.241.52/30` | `192.112.241.53` | `192.112.241.54` | Transit Link Pelabuhan |
| **`L7`** | `R-DC` <---> `R-UTARA` | `192.112.241.60/30` | `192.112.241.61` | `192.112.241.62` | Link Cadangan (File OSPF) |
| **Uplink ISP** | `R-PUSAT` <---> `R-ISP` | `203.0.113.56/30` | `203.0.113.57` | `203.0.113.58` | $203.0.113.(4 \times 14)/30$ |
| **Internet** | `R-ISP` <---> Server Internet | `8.8.8.0/24` | `8.8.8.1` | `8.8.8.8` | DNS & Internet Simulation |
| **Tunnel GRE** | `R-PUSAT` <---> `R-PELABUHAN` | `192.168.14.0/30` | `192.168.14.1` | `192.168.14.2` | $192.168.K.0/30$ |

---

## 3. Rangkuman Konfigurasi Khusus (Orang 2)

### 3.1 GRE Tunnel & Keamanan IPsec (Soal 6 - Bobot 10 + 5 Poin Bonus)
- **Interface Virtual**: `Tunnel0` dengan parameter `mtu 1476` untuk menampung *overhead* GRE 24-byte tanpa fragmentasi.
- **Rute Eksklusif**: Seluruh lalu lintas Balai Kota (`192.112.224.0/21` & `192.112.241.0/27`) menuju Pelabuhan (`192.112.160.0/19` & `192.112.240.0/25`) diarahkan secara absolut melalui IP next-hop tunnel.
- **Enkripsi IPsec over GRE**: Menerapkan IKE Phase 1 (ISAKMP Policy 10: AES-SHA-DH2, pre-shared key `K14SECURE`) dan IKE Phase 2 (Transform-Set `esp-aes esp-sha-hmac` mode transport) yang diikat pada antarmuka fisik WAN.
- *Detail analisis teknis*: Lihat [`docs/analisis_soal_6_tunnel.md`](docs/analisis_soal_6_tunnel.md).

### 3.2 Dynamic Routing OSPF & Redundansi Link L7 (Soal 7 - Bobot 10 Poin)
- **File Implementasi**: [`result/K14_VLSM_OSPF.pkt`](result/K14_VLSM_OSPF.pkt)
- **Area**: Single Area Backbone (`Area 0`).
- **Router-ID**: Diatur berurutan `1.1.1.1` (`R-PUSAT`) hingga `7.7.7.7` (`R-PELABUHAN`).
- **Passive Interface**: Seluruh antarmuka yang mengarah ke LAN switch diset `passive-interface` untuk efisiensi CPU dan keamanan dari penyusupan paket OSPF palsu.
- **Default Route Injeksi**: `R-PUSAT` menyebarkan rute default internet ke seluruh kota secara otomatis via `default-information originate`.
- **Uji Failover Link L7**: Saat Link `L1` atau `L2` diputus, rute otomatis berkonvergensi melalui Link `L7` (`192.112.241.60/30`) dalam waktu < 5 detik.
- *Detail analisis teknis*: Lihat [`docs/analisis_soal_7_ospf.md`](docs/analisis_soal_7_ospf.md).

---

## 4. DHCP Services & Layanan Aplikasi Smart City (Soal 4 & 8)

Implementasi bagian **Orang 3 (NRP 082)**. Detail teknis lengkap: [`docs/analisis_soal_4_dan_8.md`](docs/analisis_soal_4_dan_8.md).

### 4.1 DHCP Services (Soal 4)
- **36 PC pengguna** di 12 LAN kini memakai **DHCP**, menerima IP + gateway + DNS `192.112.239.2` secara otomatis.
- Router menjalankan pool DHCP per LAN (`.1-.20` di-*exclude* agar tidak bentrok dengan gateway/perangkat statis).
- **CCTV Kota dilayani DHCP server** di LAN Server & Storage DC (`192.112.239.2`), bukan oleh router:
  R-DC hanya menjadi relay (`ip helper-address`). Pool `CCTV-KOTA` = `192.112.96.100 - .119.211`.
- Router dan seluruh server tetap **IP statis**.
- Script: [`nodes/dhcp_all_routers.cfg`](nodes/dhcp_all_routers.cfg)

### 4.2 Layanan Aplikasi (Soal 8)
Domain internal: **`14.smartcity.lab`** (alias `surabaya.go.id`).

| Layanan | Server | Alamat |
|:---|:---|:---|
| DNS + DHCP | `Server dan Storage DC 1` | `192.112.239.2` |
| FTP | `Server dan Storage DC 2` | `192.112.239.3` |
| Mail (SMTP/POP3) | `Server dan Storage DC 3` | `192.112.239.4` |
| Web Portal Smart City | `Server Balai Kota 1` | `192.112.241.2` |

- **DNS**: `www` / `ftp` / `mail` → server di atas, dapat diakses dari seluruh wilayah.
- **Web**: halaman "Portal Smart City Surabaya" berisi **Kelompok 14** (`www.14.smartcity.lab`).
- **FTP**: akun `cctv` / `K14cctv` (RWDNL) agar LAN CCTV dapat mengunggah rekaman.
- **Mail**: domain `14.smartcity.lab`; akun `balai1`, `balai2`, `balai3`, `pelabuhan1`,
  `pelabuhan2`, `pelabuhan3` — teruji kirim/terima dua arah Balai Kota ↔ Pelabuhan
  (klien dikonfigurasi di seluruh PC kedua wilayah).

> ✅ **Catatan:** link **L4 (`R-PUSAT` ↔ `R-TIMUR`)** yang sebelumnya belum terpasang sudah
> diperbaiki lewat **VLAN 40 + trunk `SW-Pegawai`** (`Vlan40 192.112.241.45/30` ↔
> `R-TIMUR Gi0/0 192.112.241.46/30`). OSPF kini adjacency `FULL` dan seluruh wilayah,
> termasuk Timur, dapat mengakses internet serta layanan aplikasi. Rincian ada di
> [`docs/analisis_soal_4_dan_8.md`](docs/analisis_soal_4_dan_8.md) bagian 4.

---

## 5. Berkas Pengujian & Panduan
- **Spreadsheet Excel Resmi**: [`result/UTS-KOMDAT-JARKOM-K14.xlsx`](result/UTS-KOMDAT-JARKOM-K14.xlsx)
- **Script Konfigurasi Lengkap**: Tersedia di folder [`nodes/`](nodes/)
- **Cheat Sheet Uji 2 Individu (10 Menit)**: [`docs/cheat_sheet_uji2_orang2.md`](docs/cheat_sheet_uji2_orang2.md)
