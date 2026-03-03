# Laporan Analisis Trafik Jaringan: Magic Chess: GO GO

## 1. Pendahuluan
Laporan ini memuat hasil analisis trafik jaringan dari aplikasi mobile game yang di-capture menggunakan alat HTTP Canary (berupa file `.har`) dan PCAPdroid (berupa file `.pcap`). Tujuan analisis ini adalah untuk memetakan payload protokol antara aplikasi dan server, khususnya dimulai saat pertandingan atau *match* game dimulai.

## 2. Metodologi
Analisis dilakukan dengan menggunakan alat berbasis Command-Line Interface (CLI):
- **`jq`**: Digunakan untuk mengurai dan mengekstrak data JSON dari file `.har` (HTTP Canary).
- **`tshark`**: Digunakan untuk menganalisis dan mem-filter paket jaringan dari file `.pcap` (PCAPdroid).

## 3. Analisis `.har` (HTTP Canary)
File `hh.har` yang dihasilkan oleh HTTP Canary memberikan wawasan mengenai koneksi awal dan API analitik yang diakses oleh game.

- **Identitas Aplikasi**: Melalui file ini, terkonfirmasi bahwa nama aplikasi yang sedang dimainkan adalah **Magic Chess: GO GO** dengan *Package ID*: `com.mobilechess.gp`.
- **Domain dan Layanan Eksternal**:
  Trafik menunjukkan panggilan API ke layanan backend Moonton:
  - `publicip.mc.moontontech.com:17200/myip` (kemungkinan digunakan untuk mengambil public IP dari perangkat).
  - `logserver.msdk.moontontech.com` (layanan telemetry / pelaporan error).
  - `avatar-akm.mc.mproject.skystone.games` (aset game seperti avatar).

- **IP Server Game (Start Match)**:
  Koneksi utama yang secara spesifik menunjukkan metode `CONNECT` (koneksi websocket/tunneling) ke alamat IP server, bukan nama domain API analitik, adalah:
  1. `103.242.150.39` pada Port `14014`
  2. `8.219.151.136` pada Port `13701`

  Koneksi ke IP ini merupakan indikator kuat bahwa aplikasi sedang mencoba membangun sambungan *real-time* ke server *game match*.

## 4. Analisis `.pcap` (PCAPdroid)
Dengan menggunakan dua alamat IP server game yang didapat dari file `.har`, file `PCAPdroid_03_Mar_19_36_02.pcap` dianalisis untuk memetakan protokol saat di dalam *game match*.

### A. Protokol dan Durasi Koneksi
Hasil analisis trafik yang difilter dengan IP target `103.242.150.39` atau `8.219.151.136` memperlihatkan dominasi dua protokol:
- **UDP (User Datagram Protocol)**: ~16.588 paket (Frames)
- **TCP (Transmission Control Protocol)**: ~8.191 paket (Frames)

Ini adalah karakteristik umum dari game online *real-time*:
- TCP digunakan untuk koneksi *handshake* awal, otentikasi, atau sinkronisasi status pertandingan (handal).
- UDP digunakan untuk lalu lintas in-game secara *real-time* (gerakan, state pemain, dll) di mana kecepatan lebih penting dari pada *reliability* murni.

### B. Pemetaan Percakapan Jaringan (Network Conversations)
Dari tabel percakapan *tshark*, berikut adalah koneksi utama yang menandakan sesi *in-game*:

#### Traffic ke IP 103.242.150.39:14014
1. **UDP**:
   - Terdapat koneksi UDP masif yang berjalan selama hampir ~210 detik (sekitar 3,5 menit).
   - Pengiriman (TX): ~3451 paket / 249 kB
   - Penerimaan (RX): ~6349 paket / 353 kB
2. **TCP**:
   - Sesi TCP berdurasi ~201 detik.
   - Pengiriman (TX): ~1989 paket / 118 kB
   - Penerimaan (RX): ~2747 paket / 160 kB

#### Traffic ke IP 8.219.151.136:13701
Koneksi ke server ini lebih kecil, berjalan paralel menggunakan TCP, selama kurang lebih ~146 detik dengan total sekitar 30 kB. Ini berpotensi sebagai server penunjang atau *chat server* di dalam *match*.

### C. Alur Waktu (Timeline) Pertandingan Dimulai
Berdasarkan statistik input/output (I/O) tiap interval 10 detik, pertandingan dimulai segera di awal *capture*:
- Di 10 detik pertama, ~559 paket TCP/UDP ditransmisikan.
- Lalu lintas stabil di angka rata-rata 400 - 800 paket per 10 detik dan memakan bandwidth kurang lebih 25KB hingga 50KB setiap detiknya.
- *Match* terus berlanjut hingga detik ke-360+ (akhir tangkapan layar/capture), menandakan pertandingan berdurasi minimal 6 menit.

## 5. Analisis Mendalam Payload Protokol

Untuk memahami bagaimana aplikasi (*client*) dan server game berinteraksi di tingkat byte, dilakukan analisis *deep inspection* pada payload koneksi utama (`103.242.150.39` port `14014`).

### A. Ekstraksi dan Analisis *Magic Bytes* / Handshake
Banyak protokol game *proprietary* yang menyertakan penanda spesifik (*magic bytes*) atau versi permainan di awal koneksi. Berdasarkan ekstraksi Hex dan ASCII dari inisiasi paket UDP, ditemukan:
- **String Pengenal**: Saat string payload pertama dibalik (*reversed payload*), ditemukan teks berbunyi: `mOBILE lEGENDS.tHE bEST moba.`. Ini digunakan sebagai *handshake* awal antara client dan server untuk memverifikasi keabsahan client.
- **Versi Protokol/Game**: Ditemukan *plain-text* berupa versi build: `1.2.58.264.1G`, yang divalidasi oleh server sebelum masuk ke fase permainan.
- **Autentikasi Session**: Terdapat string yang di-*encode* Base64 berukuran besar, yang diindikasikan sebagai token sesi atau *player credential* yang dikirimkan pada awal *match*.

### B. Distribusi Ukuran Payload (Packet Size)
Game online memerlukan optimasi yang tinggi terkait *bandwidth*. Dari analisis distribusi ukuran payload data yang lewat:
- Pada **UDP** (yang merupakan tulang punggung aliran game *state*): mayoritas ukuran total data UDP di-pack sangat kecil, yaitu berukuran di bawah 50 *bytes* (akumulasi ratusan ribu kali selama *match*). Ini mencerminkan pengiriman posisi koordinat unit (*chess pieces*) atau *keep-alive pings*.
- Pada **TCP** (jalur kontrol yang reliabel): paket paling dominan adalah paket yang tidak memiliki payload data (0 byte, yang hanya berisi *ACK flag*) atau paket data kecil 20 *bytes*. Ini menegaskan TCP hanya digunakan sebagai sinyal kendali (memastikan UDP state sinkron, transaksi *item*, atau penalti koneksi/reconnect).

### C. Frekuensi Pembaruan dan Waktu antar Paket (Inter-Arrival Time / Tick-rate)
Pola waktu transmisi (*time delta* rata-rata antar paket) dalam koneksi UDP (protokol In-Game) menunjukkan:
- **Dari Server ke Aplikasi (*Downlink*)**: Paket dikirim dengan rata-rata interval **~0.021 detik**. Hal ini menunjukkan *server tick rate* berada di sekitar **47 Hz** (47 *updates* per detik).
- **Dari Aplikasi ke Server (*Uplink*)**: Paket dikirim dari perangkat pengguna dengan rata-rata interval yang sangat agresif, yakni **~0.010 detik** (~100 paket *polling*/*status report* per detik). Ini untuk memastikan responsivitas game (*zero input lag*) selama fase pertempuran *auto-battler*.

## 6. Kesimpulan
1. **Aplikasi Game**: Magic Chess: GO GO.
2. **IP Game Server (In-Match)**:
   - Server Utama: `103.242.150.39` pada port `14014`.
   - Server Pendukung: `8.219.151.136` pada port `13701`.
3. **Pola Payload dan Protokol**:
   - Game menginisiasi panggilan HTTPS via TCP ke layanan Moontontech untuk memvalidasi IP dan analitik di awal pertandingan.
   - Segera setelah otentikasi / pencarian *match* berhasil (`CONNECT` pada HTTP Canary), game membuka dua kanal: satu lewat **TCP** untuk reliabilitas state game, dan dominan lewat **UDP** untuk streaming status state permainan yang dinamis selama lebih dari 6 menit.
