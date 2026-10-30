# 🌐 Analisis Teknis Soal 7: Dynamic Routing OSPF & Redundansi Link L7 (Kelompok 14)

## 1. Arsitektur Dynamic Routing OSPF Area 0
Pada file varian kedua (`result/K14_VLSM_OSPF.pkt`), seluruh mekanisme pengalihan paket statis digantikan dengan protokol routing dinamis *link-state* **OSPFv2 (Open Shortest Path First)** dalam satu area tunggal (**Backbone Area 0**).

### A. Alokasi Router-ID Hierarkis
Untuk mempermudah pemeliharaan, pencarian kesalahan (*troubleshooting*), dan penentuan Designated Router (DR/BDR), setiap router dikonfigurasikan dengan Router-ID eksplisit:
- `R-PUSAT`: `1.1.1.1` (Core Hub Kota)
- `R-DC`: `2.2.2.2` (Data Center & Cloud Services)
- `R-BARAT`: `3.3.3.3` (Sektor Barat)
- `R-UTARA`: `4.4.4.4` (Sektor Utara & Transit Pelabuhan)
- `R-TIMUR`: `5.5.5.5` (Sektor Timur)
- `R-SELATAN`: `6.6.6.6` (Sektor Selatan)
- `R-PELABUHAN`: `7.7.7.7` (Sektor Maritim)

---

## 2. Implementasi Link Cadangan L7 (`R-DC` <---> `R-UTARA`)

### A. Alokasi Subnet
Sesuai rancangan VLSM Kelompok 14, alokasi link WAN point-to-point terakhir adalah `192.112.241.52/30` (Link L6). Oleh karena itu, Link Cadangan **`L7`** dialokasikan pada subnet `/30` berikutnya yang belum terpakai:
- **Subnet ID L7**: `192.112.241.60/30`
- **Subnet Mask**: `255.255.255.252`
- **IP Antarmuka R-DC**: `192.112.241.61`
- **IP Antarmuka R-UTARA**: `192.112.241.62`

---

## 3. Ketentuan Khusus Soal & Implementasi Teknis

### A. Kebijakan `passive-interface` ke Arah LAN Pengguna
Sesuai instruksi soal: *"Router tidak mengirim pesan OSPF ke arah LAN pengguna."*
- **Tujuan Keamanan (*Security*)**: Mencegah perangkat pengguna asing atau penyerang di jaringan LAN menyuntikkan paket OSPF palsu (*rogue OSPF neighbor*) yang dapat merusak tabel rute kota.
- **Efisiensi Sumber Daya (*Bandwidth & CPU*)**: Menghindari transmisi paket *OSPF Hello Multicast* (`224.0.0.5`) ke segmen LAN switch yang tidak memiliki router lain, sehingga menghemat *overhead* prosesor switch dan komputer klien.
- **Implementasi**: Perintah `passive-interface <interface-lan>` diterapkan di seluruh port LAN 1 dan LAN 2 di ke-7 router. Jaringan LAN tetap diiklankan ke dalam OSPF, namun router tidak mendengarkan atau mengirim paket Hello pada antarmuka tersebut.

### B. Distribusi Default Route Internet Otomatis
- **Instruksi Soal**: *"Default route ke internet hanya dikonfigurasi di R-PUSAT, dan router lain mendapatkannya secara otomatis melalui OSPF."*
- **Implementasi**: Pada `R-PUSAT` dikonfigurasikan:
  ```cisco
  ip route 0.0.0.0 0.0.0.0 203.0.113.58
  router ospf 1
   default-information originate
  ```
- **Cara Kerja**: Perintah `default-information originate` memicu `R-PUSAT` membangkitkan paket **LSA Tipe 5 (External LSA)**. Seluruh router cabang (`R-DC`, `R-UTARA`, `R-BARAT`, dsb.) akan secara otomatis menginstal rute default dengan kode:
  ```text
  O*E2 0.0.0.0/0 [110/1] via <next-hop>
  ```
  Router cabang tidak memerlukan konfigurasi static default route manual!

---

## 4. Analisis Pemulihan Jalur Otomatis (*Self-Healing Failover*)

Adanya Link Cadangan `L7` membentuk topologi cincin/segitiga redundan antara `R-PUSAT`, `R-DC`, dan `R-UTARA`.

### Skenario Uji 1: Putus Link L1 (`R-PUSAT` <-> `R-DC`)
1. **Kondisi Normal**: Trafik dari Balai Kota ke Server DC melintasi Link `L1` secara langsung (1 hop).
2. **Kondisi Gangguan (L1 Down)**: Link `192.112.241.32/30` terputus.
3. **Respon OSPF**: `R-PUSAT` dan `R-DC` mendeteksi hilangnya sinyal OSPF Hello, membangkitkan LSA Link State Update (LSU), dan menjalankan algoritma Dijkstra (Shortest Path First).
4. **Jalur Baru**: Trafik dialihkan otomatis melalui rute redundan:  
   `R-PUSAT` ➔ **Link L2** ➔ `R-UTARA` ➔ **Link L7** ➔ `R-DC` (Server & Storage DC).
5. **Waktu Konvergensi**: < 5 detik, tanpa ada intervensi manual administrator.

### Skenario Uji 2: Putus Link L2 (`R-PUSAT` <-> `R-UTARA`)
1. **Kondisi Normal**: Trafik dari Balai Kota ke Pelabuhan melintasi Link `L2` lalu Link `L6`.
2. **Kondisi Gangguan (L2 Down)**: Kabel `192.112.241.36/30` terputus.
3. **Respon OSPF**: Jalur ke Sektor Utara dan Pelabuhan dialihkan secara dinamis melalui:  
   `R-PUSAT` ➔ **Link L1** ➔ `R-DC` ➔ **Link L7** ➔ `R-UTARA` ➔ `R-PELABUHAN`.
4. **Hasil**: Komunikasi SCADA dan Kantor Pelabuhan tetap berjalan stabil dan tidak terisolasi.
