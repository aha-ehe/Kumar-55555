# Deep Dive Analysis: Reverse Engineering Payload Game Magic Chess: GO GO

Laporan ini mendokumentasikan hasil analisis teknis pada tingkat biner (raw hex) dari transmisi protokol jaringan antara klien (aplikasi mobile) dan server game (103.242.150.39:14014).

## 1. Paket Inisiasi UDP (Handshake Klien -> Server)
Paket pertama yang dikirim dari klien berukuran 52 bytes. Paket ini berfungsi sebagai *hello/handshake* awal.

**Hex Dump:**
```hex
01 71 5f e5 63 3f 35 96 27 c2 db 90 2c 03 2e 61
62 6f 6d 20 54 53 45 62 20 45 48 74 2e 53 44 4e
45 47 45 6c 20 45 4c 49 42 4f 6d 01
```

**Analisis Struktur:**
- `01 71`: Kemungkinan besar merupakan Opcode / Packet ID yang menandakan "Client Hello".
- `5f e5 63 3f ...`: Payload acak atau Session ID awal (12 bytes).
- `2e 61 ... 6d 01`: *Magic Bytes* / *Plaintext* string yang ditulis secara terbalik (*Little Endian string representation* atau sekadar *reversed array*). Jika dibalik (reverse), hex tersebut akan menjadi string: **`mOBILE lEGENDS.tHE bEST moba.`**. String ini adalah *fingerprint* khas dari Moonton / Mobile Legends engine yang digunakan oleh game ini.

## 2. Payload UDP Keep-Alive / State Update (Klien -> Server)
Setelah koneksi terjalin, klien mengirim paket UDP berukuran 36-38 bytes setiap ~0.010 detik secara terus-menerus.

**Hex Dump:**
```hex
01 75 35 13 68 14 14 00 70 00 e2 08 01 a3 1c 45
05 70 00 8e 1e 80 06 01 07 8e 1e 80
```

**Analisis Struktur:**
- `01 75`: Opcode yang menandakan "State Update" atau "Ping/Keep-Alive".
- `35 13 68 14`: Kemungkinan adalah *Client ID*, *Match ID*, atau *Timestamp* (terlihat statis dalam *session* yang sama).
- `14 00` (Hex) = `20` (Decimal). Jika ditambah header (16 bytes), cocok dengan panjang sisa data dalam payload. Ini adalah **Payload Length** marker.
- `70 00 e2 ...`: Koordinat biner / status in-game yang di-encode.

## 3. Jalur Kontrol TCP (Server -> Klien)
Selain UDP, ada sesi TCP yang berjalan paralel untuk kontrol yang lebih reliabel (seperti chat room, sinkronisasi *Voice Room*, atau data pemain yang krusial).

**Hex Dump Respons Besar TCP (625 bytes):**
```hex
00 00 02 71 70 00 ea 07 46 e5 04 70 40 8b 01 30
30 36 34 36 63 64 32 37 64 64 62 33 66 38 34 39
36 34 39 31 34 61 38 38 37 64 36 37 33 39 31 36
... (truncated)
```

**Analisis Struktur TCP:**
- `00 00 02 71`: 4 byte pertama adalah **Header Panjang Paket (Packet Length)**. Hex `02 71` sama dengan panjang 625 desimal. Ini sangat umum dalam implementasi TCP untuk memecahkan masalah *TCP Stream Framing*.
- **Plaintext / Meta-Data Ditemukan dalam Paket:**
  - `00646cd27ddb3f84964914a887d6739165b` (Hash Session ID).
  - String JSON/Base64 Padding: `IACyeidv2NBvKuV8bR6XAa...` (Kemungkinan besar *Player Access Token* atau *Authentication key* dari platform Moonton).
  - Identifier Pemain: `00167515765f649ea018bcd8949`
  - Identifier Room Pertandingan: **`Play_428266064328447023_428266064328447023C`** dan **`Play_RoomVoice_3739671916`**. Ini membuktikan bahwa jalur TCP digunakan untuk membentuk saluran suara (*Voice Chat*) serta sesi *lobby/matchmaking* secara reliabel, sementara game statenya diserahkan ke UDP.

## 4. Kesimpulan Struktur Biner
1. **Header TCP**: Menggunakan 4-byte *Length Prefix* di awal tiap pengiriman, membuktikan engine jaringan ini standar (seperti *Netty* atau implementasi *socket* C++ kustom).
2. **Obfuscation**: Moonton menggunakan trik dasar seperti membalik string (*reversed text*) pada *handshake* awal UDP, namun data krusial seperti Token Sesi dan *Room ID* ("Play_RoomVoice_...") ditransmisikan dalam bentuk Base64/Plain-text tanpa enkripsi TLS (terbaca langsung dari packet payload TCP).
3. **Pemisahan Trafik**: Game mendelegasikan perintah penting/state room ke TCP (terbukti dari adanya string `RoomVoice` dan Auth Key), dan koordinat pertempuran cepat (tick-rate tinggi) ke UDP dengan payload biner yang sangat padat (36 bytes).
