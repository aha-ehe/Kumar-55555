# Security & Vulnerability Analysis: Magic Chess: GO GO

Berdasarkan *traffic capture* (via file `.pcap` dan `.har`), analisis ini berfokus pada potensi kerentanan keamanan dan risiko eksploitasi protokol kustom yang digunakan oleh aplikasi dalam berkomunikasi dengan server *matchmaking* dan server *in-game*.

## 1. Ringkasan Ancaman (Threat Summary)
Protokol game menggunakan HTTP/HTTPS pada tahap awal (*lobby* dan analitik), namun beralih sepenuhnya ke transmisi *plaintext* (TCP/UDP socket murni) pada saat fase krusial: pertempuran (*match*). Karena kurangnya enkripsi tingkat transpor (Transport Layer Security / TLS) selama in-game, protokol ini **rentan terhadap berbagai serangan jaringan pasif maupun aktif**.

## 2. Temuan Keamanan Utama

### A. Pengiriman Data Sensitif Tanpa Enkripsi (Insecure Data Transmission)
**Deskripsi Temuan:**
Ekstraksi payload jaringan (via de-obfuscation) memperlihatkan bahwa statistik pertempuran, state unit, status HP, koordinat pemain, dan token otentikasi sesi dikirim **tanpa enkripsi** yang kuat. Data game (*game state*) seperti:
`client_MagicchessRoundHeroInfo|428266064328447023|8|91914637|36|...`
terlihat jelas dalam format *pipeline-delimited* (`|`). Selain itu, string sesi Voice/Chat room (`Play_RoomVoice_3739671916`) mengalir lewat TCP telanjang.

**Risiko (Risk):**
- **Passive Sniffing (Penyadapan)**: Pihak ketiga di jaringan publik (seperti Wi-Fi cafe) dapat membaca *traffic* ini untuk mengetahui informasi pribadi (seperti hash akun) dan state permainan.
- **Drop Hacking / Connection Spoofing**: Penyerang bisa membaca *Sequence Number* dari paket TCP dan mengirim paket *Reset* (RST) palsu untuk memutuskan koneksi pemain target (*Drop hack*).

### B. Kelemahan Obfuscation Mekanisme Handshake (Security by Obscurity)
**Deskripsi Temuan:**
Satu-satunya bentuk "perlindungan" yang terlihat untuk menutupi *magic bytes* atau paket tertentu adalah rotasi byte sederhana atau pembalikan urutan teks (reverse text). Contoh: string otentikasi klien `mOBILE lEGENDS.tHE bEST moba.` diputar 180 derajat (`.abom TSEb EHt.SDNEGEl ELIBOm`).

**Risiko (Risk):**
- Teknik ini (*Security by Obscurity*) sangat mudah di-*reverse engineer* (dibongkar).
- Membuka ruang besar untuk pembuatan **Cheat Tool / Maphack**. Pembuat cheat tidak perlu repot mencari kunci dekripsi di dalam *memory* game (RAM), mereka cukup membuat *proxy* jaringan yang membaca *traffic* mentah, mem-parsing format `|` delimiternya, lalu menampilkan *dashboard* statistik lawan (seperti posisi musuh, unit *Magic Chess* yang dibeli lawan, formasi, dll) di layar terpisah (Radar Hack via Network).

### C. Kurangnya Anti-Replay pada Sesi Non-TLS (Replay Attack Vulnerability)
**Deskripsi Temuan:**
Karena transmisi UDP dan TCP ini kustom dan tidak berada di bawah kanopi TLS, ada potensi bahwa paket otentikasi awal (yang berisi token *login match*) tidak memiliki mekanisme *nonce* (nomor acak yang unik per sesi) atau validasi timestamp yang mengikat kriptografis.

**Risiko (Risk):**
- **Replay Attack**: Seseorang bisa merekam (*sniff*) paket persetujuan (ACK) dari item yang dibeli / kemampuan yang diaktifkan, dan mengirimkannya ulang (*replay*) berulang-ulang ke server, berpotensi memicu *glitch* state (seperti spam pembelian item/uang palsu).

## 3. Rekomendasi Mitigasi (Bagi Developer)
1. **Penerapan Enkripsi Tingkat Socket**: Mengganti *socket* TCP/UDP telanjang dengan implementasi DTLS (Datagram Transport Layer Security) untuk UDP, dan TLS murni untuk TCP. Ini akan menghapus vektor serangan *sniffing* secara instan.
2. **Kriptografi Asimetris untuk Handshake**: Daripada sekadar membalik string statis `mOBILE lEGENDS...`, server harus mengirim *challenge nonce* yang ditandatangani oleh klien menggunakan kunci rahasia (*secret key*) yang tersemat (obfuscated) di dalam binary game (seperti JNI lib di Android).
3. **Penyandian Biner (Binary Packing)**: Mengganti format komunikasi *pipeline delimiter* (string) `|` yang sangat boros *bandwidth* dan mudah dibaca, dengan protokol pemaketan biner efisien seperti Protobuf (Protocol Buffers) atau FlatBuffers, ditambah lapisan enkripsi (seperti AES-GCM) berkinerja tinggi.
