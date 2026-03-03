# Flow Analysis: Alur Matchmaking Magic Chess: GO GO

Analisis korelasi antara *timestamp* di file `.har` (HTTP/HTTPS) dan *relative time* di file `.pcap` (UDP/TCP) mengungkapkan alur masuk (onboarding) game dari *lobby* menuju ke *in-match*.

## Timeline Sinkronisasi (Berdasarkan Waktu)
Waktu *capture* diasumsikan beririsan dekat antara file log Canary dan PCAPdroid.

**1. T0 - Inisiasi Layanan & Telemetri**
- Game melakukan pemanggilan ke `logserver.msdk.moontontech.com/api/v1/report-compact`.
- Request ini membawa `token` enkripsi (base64) untuk melaporkan statistik pengguna bahwa mereka siap memulai permainan atau sudah menyelesaikan proses otentikasi lobi awal.
- Pemanggilan API dilakukan berulang (retries).

**2. T0 + ~1 Detik - Validasi Jaringan Eksternal**
- Game melakukan GET request ke `publicip.mc.moontontech.com:17200/myip`.
- *Tujuan*: Game server Moonton perlu memastikan alamat IP publik klien yang sebenarnya untuk menyiapkan alokasi NAT *hole-punching* (krusial untuk koneksi UDP *peer-to-server*).

**3. T0 + ~2 Detik - Inisiasi Koneksi Penunjang**
- Klien melakukan `CONNECT` ke alamat IP `8.219.151.136:13701`.
- Dalam file pcap, sesi TCP ini mulai berjalan dan bertahan selama kurang lebih 146 detik (berada dalam status koneksi terbuka namun transmisi data relatif kecil, ~30KB). Alur ini bertindak sebagai *Voice/Chat Lobby* atau *Auxiliary Server*.

**4. T0 + ~10 Detik - Match Found & Handover (Momen Kritis)**
- Terdapat jeda beberapa detik di mana klien menunggu respons *matchmaking* dari server lobi.
- Pada detik ke-10 (T0 + 10s / `12:29:37.774Z` di `.har`), HTTP Canary mencatat koneksi *Tunnel/CONNECT* ke target IP server arena: **`103.242.150.39:14014`**.
- Hampir bersamaan di file `.pcap` (sekitar *relative time* 10 detik / 159 detik tergantung referensi start), trafik TCP meledak menuju server ini dan *stream* TCP dipertahankan untuk mengunci status ruangan (Room State).

**5. T0 + ~10.5 Detik - Inisiasi UDP Game Loop (In-Match)**
- Segera setelah *stream* TCP berhasil (*handshake* diterima dan token Room diverifikasi), mesin game membuka *port* UDP.
- Klien mengirim payload `mOBILE lEGENDS.tHE bEST moba.` ke server via UDP port 14014.
- Server merespons (Tick rate 47Hz dimulai) dan data koordinat pertempuran mulai membanjiri jalur jaringan. Pertandingan resmi dimulai.

## Kesimpulan Alur Matchmaking
Alur masuk ke game cukup standar untuk *esports/competitive title*:
`Telemetri (Lobby) -> Cek Public IP (NAT Info) -> Koneksi Chat Room (TCP) -> Match Found (Handover via HTTP CONNECT) -> Room Session Auth (TCP) -> Fast-paced Game Loop (UDP).`
