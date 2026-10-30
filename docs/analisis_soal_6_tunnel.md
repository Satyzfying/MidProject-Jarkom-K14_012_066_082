# 🔒 Analisis Teknis Soal 6: GRE Tunnel & Keamanan IPsec (Kelompok 14)

## 1. Konsep & Arsitektur Tunneling
Untuk menghubungkan dua pusat administratif strategis Pemerintah Kota Surabaya (Balai Kota dan Pelabuhan), dibangun jalur privat virtual **Generic Routing Encapsulation (GRE) Tunnel** menembus jaringan backbone perantara (`R-UTARA`).

- **Subnet Virtual Tunnel**: `192.168.14.0/30`
- **Ujung Tunnel R-PUSAT**: `192.168.14.1` (Underlay IP Fisik: `192.112.241.37`)
- **Ujung Tunnel R-PELABUHAN**: `192.168.14.2` (Underlay IP Fisik: `192.112.241.54`)

---

## 2. Mengapa Router Memilih Jalur Tunnel Dibandingkan Jalur Fisik Lain?

Dalam arsitektur jaringan Cisco IOS, keputusan pemilihan jalur paket ditentukan oleh hierarki algoritma *decision-making*:

### A. Prinsip *Longest Prefix Match* (Prioritas Utama)
Tabel routing selalu memprioritaskan entri rute dengan subnet mask paling panjang/spesifik:
1. **Di `R-PUSAT`**:
   - Rute umum menuju arah utara/internet adalah default route `0.0.0.0/0` (prefix `/0`, paling umum) atau subnet antar-link `192.112.241.52/30`.
   - Rute statis yang kita pasang mengarah langsung ke subnet spesifik:
     - SCADA Pelabuhan: `192.112.160.0/19` via `192.168.14.2`
     - Kantor Pelabuhan: `192.112.240.0/25` via `192.168.14.2`
   - Karena prefix `/19` dan `/25` jauh lebih spesifik daripada default route `/0`, router secara absolut memilih antarmuka `Tunnel0`.
2. **Di `R-PELABUHAN`**:
   - Rute keluar bawaan hanya memiliki default route `0.0.0.0/0` ke `R-UTARA`.
   - Rute menuju LAN Balai Kota (`192.112.224.0/21` dan `192.112.241.0/27`) diarahkan ke next-hop `192.168.14.1`.
   - Sesuai prinsip *longest prefix match*, paket ke Balai Kota otomatis dialihkan masuk ke dalam tunnel.

### B. Evaluasi *Administrative Distance* (AD)
- Pada implementasi file statis maupun OSPF, rute statis yang menunjuk ke interface tunnel memiliki nilai **Administrative Distance = 1**.
- Jika protokol dinamis OSPF aktif dan mempelajari rute fisik alternatif lewat `R-UTARA`, rute OSPF tersebut memiliki **AD = 110**.
- Rute statis (AD 1) selalu mengungguli rute OSPF (AD 110) dalam proses instalasi ke RIB (*Routing Information Base*).

---

## 3. Mengapa GRE Saja Tidak Mengenkripsi Data?

### A. Hakikat Protokol GRE (IP Protocol 47)
GRE diciptakan murni sebagai protokol **enkapsulasi**, bukan protokol keamanan:
- GRE membungkus (*encapsulate*) paket data IP internal dengan menambahkan header GRE (4 byte) dan header IP luar (20 byte).
- **Tidak ada mekanisme enkripsi**: Isi muatan paket (*payload*) di dalam paket GRE tetap berupa teks mentah (*plaintext*).
- **Tidak ada autentikasi & integritas**: GRE tidak memiliki *hash message authentication* (HMAC) untuk memverifikasi apakah paket telah dimodifikasi di tengah jalan oleh pihak ketiga.

### B. Bukti Kerentanan pada Simulasi
Jika paket data melintasi router perantara (`R-UTARA`), administrator atau penyusup di `R-UTARA` dapat menginspeksi lalu lintas (misalnya menggunakan *Simulation Mode / Packet Sniffer* di CPT) dan membaca secara langsung seluruh isi dokumen, kredensial, maupun perintah kontrol SCADA pelabuhan.

### C. Solusi: Enkripsi IPsec over GRE (Bonus +5 Poin)
Untuk mengatasi kelemahan mendasar GRE tersebut, kami menerapkan arsitektur **IPsec over GRE**:
1. **IKE Phase 1 (ISAKMP Policy)**: Membentuk saluran negosiasi aman menggunakan algoritma simetris `AES`, fungsi hash `SHA`, dan pertukaran kunci `Diffie-Hellman Group 2` dengan pre-shared key `K14SECURE`.
2. **IKE Phase 2 (Transform-Set)**: Menerapkan protokol **ESP** (*Encapsulating Security Payload*) dengan enkripsi `esp-aes` dan integritas `esp-sha-hmac` dalam mode **transport**.
3. **Crypto Map**: Seluruh paket berprotokol GRE (IP proto 47) antara IP fisik `192.112.241.37` dan `192.112.241.54` otomatis dienkripsi sebelum meninggalkan kartu antarmuka fisik, sehingga data dijamin 100% rahasia saat melewati `R-UTARA`.

---

## 4. Parameter MTU 1476
Enkapsulasi GRE menambahkan *overhead* sebesar 24 byte pada paket IP standar (MTU 1500). Untuk mencegah terjadinya fragmentasi paket (*packet fragmentation*) yang dapat menurunkan performa atau menyebabkan *packet drop*, antarmuka `Tunnel0` dikonfigurasi dengan `mtu 1476` ($1500 - 24 = 1476$).
