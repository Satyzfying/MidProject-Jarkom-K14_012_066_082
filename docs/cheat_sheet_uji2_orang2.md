# 🚨 Cheat Sheet Uji 2 Individu (Materi Orang 2: Tunnel & OSPF)
**Waktu Ujian: 10 Menit | Bobot: 15 Poin | Target: Temukan, Perbaiki, & Jelaskan**

---

## 🛠️ Perintah Sakti Pertama Saat Duduk di Depan Laptop:
Ketik 3 perintah ini dalam 30 detik pertama:
```cisco
enable
show ip interface brief           ! Cek port mana yang sengaja di-shutdown atau salah IP
show ip route                     ! Cek apakah ada rute O (OSPF) atau S (Static)
show ip ospf neighbor             ! Cek apakah tetangga OSPF statusnya 'FULL'
```

---

## 💥 Skenario 1: GRE Tunnel Down / Tidak Bisa Ping Balai Kota ➔ Pelabuhan

### 1. Gejala:
Ping dari Server Balai Kota (`192.112.241.2`) ke SCADA Pelabuhan (`192.112.160.2`) gagal / Request Timed Out, atau `show interfaces tunnel 0` menunjukkan:  
`Tunnel0 is up, line protocol is down`

### 2. Lokasi Pengecekan:
Ketik di `R-PUSAT` dan `R-PELABUHAN`:
```cisco
show run interface tunnel 0
```

### 3. Kemungkinan Sabotase Aslab & Cara Memperbaikinya:
| Sabotase yang Sering Dibuat Aslab | Cara Mengetahuinya | Solusi Perbaikan Cepat (CLI) |
|---|---|---|
| **1. Tunnel Destination Salah IP** | IP destination diisi IP LAN, bukan IP fisik WAN seberang | `interface Tunnel0`<br>`tunnel destination 192.112.241.54` *(di R-PUSAT)*<br>`tunnel destination 192.112.241.37` *(di R-PELABUHAN)* |
| **2. Interface Tunnel di-Shutdown** | Tulisan `administratively down` | `interface Tunnel0`<br>`no shutdown` |
| **3. Subnet Tunnel Beda** | IP tunnel bukan pasangan `/30` | Samakan: R-PUSAT = `192.168.14.1 255.255.255.252`, R-PELABUHAN = `192.168.14.2 255.255.255.252` |
| **4. Rute Statis Tunnel Hilang** | Cek `show ip route static` | Pasang rute: `ip route 192.112.160.0 255.255.224.0 192.168.14.2` |

---

## 💥 Skenario 2: OSPF Neighbor Tidak Terbentuk / Rute OSPF Hilang

### 1. Gejala:
`show ip ospf neighbor` kosong (tidak ada tetangga), atau rute bertanda `O` tidak muncul di tabel routing.

### 2. Kemungkinan Sabotase Aslab & Cara Memperbaikinya:
| Sabotase Aslab | Cara Mengetahuinya | Solusi Perbaikan Cepat (CLI) |
|---|---|---|
| **1. Port WAN dibuat `passive-interface`** *(Paling Sering!)* | Cek `show run \| section ospf`, ada `passive-interface` pada port antar-router | Masuk ke router ospf dan matikan:<br>`router ospf 1`<br>`no passive-interface <nama_port_wan>` |
| **2. Beda Area ID** | Di satu router tertulis `area 0`, di router sebelah tertulis `area 1` | Ubah menjadi area yang sama:<br>`router ospf 1`<br>`network <net> <wildcard> area 0` |
| **3. Wildcard Mask Salah** | Misal subnet `/30` ditulis wildcard `0.0.0.255` bukan `0.0.0.3` | Hapus dan benahi:<br>`no network ...`<br>`network 192.112.241.x 0.0.0.3 area 0` |
| **4. Hello/Dead Timer Diubah** | Cek `show ip ospf interface <port>` | Hapus timer custom di interface:<br>`interface <port>`<br>`no ip ospf hello-interval`<br>`no ip ospf dead-interval` |
| **5. Default Route Internet Hilang di Cabang** | Router cabang tidak punya rute `O*E2 0.0.0.0/0` | Buka `R-PUSAT`:<br>`router ospf 1`<br>`default-information originate` |

---

## 💥 Skenario 3: Link Cadangan L7 Gagal Failover Saat Kabel Diputus

### 1. Gejala:
Saat kabel Link L1 atau L2 dihapus/diputus di CPT, ping ke DC atau Utara langsung putus dan tidak kembali nyambung.

### 2. Langkah Cepat Menemukan Masalah:
1. Periksa interface L7 di `R-DC` dan `R-UTARA`:
   ```cisco
   show ip interface brief
   ```
   *Apakah port L7 statusnya DOWN atau SHUTDOWN?* ➔ Ketik `no shutdown`.
2. Periksa apakah subnet L7 sudah diiklankan di OSPF:
   ```cisco
   show run | section ospf
   ```
   *Pastikan ada baris:*
   `network 192.112.241.60 0.0.0.3 area 0` di **`R-DC`** DAN di **`R-UTARA`**.
3. Pastikan IP di kedua sisi sinkron:
   - `R-DC`: `192.112.241.61 255.255.255.252`
   - `R-UTARA`: `192.112.241.62 255.255.255.252`

---

## 🗣️ Cara Menjelaskan ke Dosen (Template Jawaban 30 Detik):
> *"Izin menjelaskan Pak/Bu, kesalahan tadi berada pada [nama router], di mana antarmuka [nama port] statusnya [misal: di-passive-interface / salah IP destination tunnel]. Akibatnya, paket OSPF Hello tidak dapat terkirim sehingga adjacency neighbor gagal terbentuk / rute tidak terinstal di routing table. Saya telah memperbaikinya dengan [sebutkan perintahmu], dan saat ini status neighbor sudah kembali 'FULL' serta seluruh rute telah pulih normal seperti yang terlihat pada hasil ping ini."*
