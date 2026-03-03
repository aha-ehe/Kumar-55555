# Analisis Kerentanan Denial of Service (DoS): Magic Chess: GO GO

Berdasarkan data *traffic capture* (`.pcap`) yang telah dibedah pada tahap sebelumnya, protokol kustom (TCP/UDP) yang digunakan oleh server In-Game (`103.242.150.39:14014`) menunjukkan beberapa celah arsitektur jaringan yang secara teoretis memungkinkan terjadinya serangan **Denial of Service (DoS)** atau *Room Crashing* (menghancurkan/membuat *lag* satu ruangan pertandingan).

Berikut adalah 3 vektor serangan DoS utama yang ditemukan:

## 1. Kerentanan Alokasi Memori TCP (TCP Length Prefix Exploitation)
Pada laporan `payload_analysis.md`, ditemukan bahwa koneksi TCP game menggunakan format *4-byte Length Prefix Header* (misalnya: `00 00 02 71` yang mengindikasikan bahwa sisa data/payload adalah 625 bytes).
- **Celah:** Karena TCP adalah aliran kontinu (*stream*), *server* game (*buffer parser*) harus membaca 4-byte pertama, mengalokasikan memori (*RAM/Buffer*) di memori server sesuai angka tersebut, lalu menunggu sisa data terkirim.
- **Skenario Serangan (Memory Exhaustion / Slowloris):** Seorang penyerang dapat membuka banyak koneksi TCP ke `103.242.150.39:14014`, mengirimkan *header* yang diklaim sangat besar (misalnya `FF FF FF FF` = ~4 GB), dan perlahan-lahan mengirim sisa datanya se-byte demi se-byte. Tanpa batasan ukuran maksimum (*Max Frame Size Limit*) atau penanganan *timeout* yang agresif di level aplikasi (Netty/C++ socket server), server *match* akan kehabisan *heap memory* hanya untuk menampung koneksi palsu ini, menyebabkan seluruh pemain di server tersebut terputus (Disconnect).

## 2. Serangan Kelelahan CPU (CPU Exhaustion) via Payload Malformasi
Skrip ekstraksi (*deobfuscation*) menemukan bahwa struktur pembaruan status pahlawan (*Hero Status*) menggunakan format data *piping/CSV-style* yang sangat panjang dalam format *plaintext*:
`client_MagicchessRoundHeroInfo|428266064328447023|8|91914637|36|...|53:2,6;2:2,6;58:2,2;...`

- **Celah:** Pemrosesan string murni (`|` dan `;` delimiter parsing) membutuhkan daya komputasi CPU (Regex/Split) yang jauh lebih tinggi daripada parsing biner terkompresi. Tidak adanya enkripsi TLS berarti penyerang bisa mengirim format ini secara sewenang-wenang.
- **Skenario Serangan (Payload Bomb):** Penyerang dapat membuat program/skrip yang membanjiri server dengan jutaan baris string `client_MagicchessRoundHeroInfo` di mana pembatas `|` atau koordinat `;` dibuat bersarang atau rusak berukuran puluhan Kilobyte per paket. Algoritma pemisah string (Splitter) di *backend* server berpotensi terjebak dalam *looping* pemrosesan berat (*Regex DoS* atau alokasi *garbage collection*), menyebabkan *tick-rate* server (*47Hz*) anjlok (Lag/Spike) untuk seluruh *room*.

## 3. UDP Flood dan Amplifikasi Klien (IP Spoofing)
Protokol utama pergerakan unit dilakukan melalui koneksi *UDP stateless* (tanpa DTLS).
- **Celah:** Klien mengirimkan pembaruan status (Uplink) secara sangat agresif setiap ~0.010 detik (~100 paket/detik).
- **Skenario Serangan (Room State Flooding):** Penyerang bisa memanfaatkan ini dengan menulis *script* Python sederhana yang meniru *Magic Bytes* awal (`mOBILE lEGENDS.tHE bEST moba.`), dan menyalin (replay) ID Sesi (*Room ID*). Karena paket ini UDP tanpa penanda waktu kriptografis (*nonce/timestamp sequence* aman), penyerang bisa melipatgandakan pengiriman paket pergerakan (ratusan ribu paket palsu per detik) yang ditujukan ke *port* target. Hal ini tidak hanya membebani server karena memproses status pahlawan fiktif, tetapi paket yang *invalid* ini bisa memicu *crash* internal *game engine* (kondisi *race/out-of-bounds array*).

## Kesimpulan
Desain protokol jaringan game yang mengandalkan transmisi biner telanjang (Plaintext TCP & UDP) dengan string parsing panjang (`|`), **sangat rawan terhadap serangan DoS pada skala aplikatif (Layer 7)**. Penyerang tidak butuh *Botnet* berukuran besar (seperti DDoS volumetrik Layer 4); cukup 1 PC dengan skrip Python (*Low-and-Slow Attack* atau *Payload Bomb*) untuk merusak kestabilan satu ruangan pertandingan (*Match/Room Crashing*).
