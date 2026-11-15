# Cheat Sheet Uji 2 Individu (Materi Orang 1: Static Routing & NAT)

**Waktu Ujian: 10 Menit | Bobot: 15 Poin | Target: Temukan, Perbaiki, & Jelaskan**

---

## Perintah Sakti Pertama

```cisco
enable
show ip interface brief
show ip route
show ip nat translations
show ip nat statistics
```

---

## Skenario 1: PC Tidak Bisa Ping Gateway / Antar-LAN

### Gejala
Ping dari PC ke gateway gagal, atau antar-LAN tidak tembus.

### Lokasi Pengecekan
- `show ip interface brief` di router cabang dan `R-PUSAT`.
- `show ip route` di router sumber.

### Sabotase yang Sering Dibuat Aslab

| Sabotase | Cara Mengetahui | Perbaikan Cepat |
|---|---|---|
| Interface LAN di-shutdown | Status `administratively down` | `interface <port>` lalu `no shutdown` |
| IP gateway salah | IP interface tidak sesuai tabel VLSM | `ip address <gateway> <mask>` |
| Static route cabang hilang | Tidak ada rute `S` ke LAN tujuan | `ip route <network> <mask> <next-hop>` |
| Default route cabang hilang | Tidak ada `S* 0.0.0.0/0` | `ip route 0.0.0.0 0.0.0.0 <IP R-PUSAT>` |
| Rute balik di `R-PUSAT` hilang | Ping satu arah | Tambahkan rute ke LAN cabang via IP WAN cabang |

---

## Skenario 2: Internet Gagal / NAT Tidak Jalan

### Gejala
Ping ke `8.8.8.8` gagal dari PC, tetapi ping ke gateway berhasil.

### Lokasi Pengecekan
Di `R-PUSAT`:

```cisco
show ip route
show ip nat statistics
show ip nat translations
show running-config | include nat
```

### Sabotase yang Sering Dibuat Aslab

| Sabotase | Cara Mengetahui | Perbaikan Cepat |
|---|---|---|
| `ip nat outside` hilang di uplink ISP | Interface `Gi0/0/2` tidak bertanda outside | `interface GigabitEthernet0/0/2` lalu `ip nat outside` |
| `ip nat inside` hilang di LAN | Interface Vlan tidak bertanda inside | `interface Vlan21` (dan Vlan lain) lalu `ip nat inside` |
| ACL NAT salah | `show access-lists` tidak permit `192.112.0.0/16` | `access-list 1 permit 192.112.0.0 0.0.255.255` |
| NAT overload salah interface | `ip nat inside source list` menunjuk interface salah | `ip nat inside source list 1 interface GigabitEthernet0/0/2 overload` |
| Default route `R-PUSAT` hilang | Tidak ada `S* 0.0.0.0/0` | `ip route 0.0.0.0 0.0.0.0 203.0.113.58` |

---

## Skenario 3: R-ISP Punya Rute Privat

### Gejala
Soal meminta `R-ISP` tidak memiliki rute ke blok privat, tetapi `show ip route` menampilkan `192.112.0.0/16`.

### Perbaikan

```cisco
enable
configure terminal
no ip route 192.112.0.0 255.255.0.0 <next-hop>
end
write memory
```

Pastikan `R-ISP` hanya memiliki rute default ke `R-PUSAT` dan rute internet `8.8.8.0/24`.

---

## Cara Menjelaskan ke Dosen (30 Detik)

> "Izin menjelaskan, Pak/Bu. Masalahnya ada di [nama router], pada [interface/route/NAT]. Akibatnya paket [tidak keluar / tidak kembali / tidak ter-NAT]. Saya perbaiki dengan [perintah], lalu saya verifikasi lewat `show ip route` dan ping ke `8.8.8.8`; hasilnya sudah normal."
