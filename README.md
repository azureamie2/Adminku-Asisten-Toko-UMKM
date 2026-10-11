# AdminKu: Asisten Admin AI untuk Toko UMKM

AdminKu adalah asisten admin berbasis AI untuk pemilik toko online skala kecil. Dalam satu antarmuka, pemilik toko bisa mendapat **draf balasan pelanggan**, **caption produk**, dan **rekap penjualan**, ditambah kalkulator pesanan.

Studi kasus: **Dapur Bu Rina**, toko camilan dan kue kering (data contoh fiktif).

Dibuat untuk Hackathon Hacktiv8 | Tema: **Productivity & Smart Business**

---

## Masalah

Pemilik UMKM makanan online yang mengelola toko sendirian menerima pertanyaan pelanggan yang sama berulang kali: harga, ongkir, cara pesan. Informasinya tersimpan di ingatan atau catatan tidak terstruktur, sehingga balasan sering telat dan bisa berbeda antar pelanggan. Caption produk dan rekap penjualan juga masih dikerjakan manual. Waktu yang habis untuk admin adalah waktu yang hilang dari produksi.

## Solusi

- **Jawaban dari data toko sendiri.** Agent wajib mencari di katalog dan FAQ. Jika info tidak ada, ia menjawab bahwa akan dicek ke pemilik, tidak menebak harga, stok, atau kebijakan.
- **Pemilik tetap mengendalikan.** Semua balasan pelanggan berupa draf yang ditinjau dulu sebelum dikirim.
- **Akses data hanya baca.** Agent membaca Google Sheets tanpa bisa mengubahnya, dan angka rekap berasal dari spreadsheet, bukan dihitung ulang oleh AI.
- **Satu tempat untuk tiga pekerjaan admin**, dengan bahasa Indonesia sehari-hari.

## Fitur Utama

| Fitur | Fungsi |
|---|---|
| Draf Balasan Pelanggan | Mencari jawaban di katalog dan FAQ (Astra DB), lalu menyusun draf balasan |
| Caption Produk | Menulis caption dan deskripsi dari data produk yang diketik pemilik, tanpa menambah klaim yang tidak disebutkan |
| Rekap Penjualan | Membaca ringkasan dari Google Sheets: total, produk terlaris, omzet per minggu |
| Hitung Pesanan | Kalkulator di antarmuka web untuk total bayar (barang, ongkir, diskon, DP) dan ringkasan pesanan siap salin. Berjalan di web sendiri tanpa Langflow |

## Arsitektur

```
Pemilik toko
    │  (mengetik permintaan)
    ▼
Antarmuka web (index.html)
    │  POST /api/v1/run/{flow_id}
    ▼
IBM Langflow ── Agent AdminKu
    ├── Tool Astra DB       → katalog & FAQ (RAG)
    └── Tool Google Sheets  → rekap penjualan (hanya baca)
    │
    ▼
Draf jawaban → pemilik meninjau → dikirim ke pelanggan
```

### Flow Langflow

Satu flow bernama **"AdminKu - Asisten Toko UMKM"** dengan dua bagian:

1. **Ingest data:** `Read File` → `Split Text` (chunk 1000, overlap 200) → `Astra DB` (collection `dapur`, embedding Google Generative AI)
2. **Percakapan:** `Chat Input` → `Agent` → `Chat Output`. Agent memiliki dua tool: `Astra DB` (similarity search, 4 hasil) dan `Google Sheets` (Composio)

Aturan utama di System Prompt: wajib mencari di Astra DB sebelum menjawab info toko, tidak mengarang harga, stok, atau kebijakan, semua balasan pelanggan berstatus draf, Google Sheets hanya boleh dibaca, dan pertanyaan di luar topik toko ditolak dengan sopan.

### Peran IBM Bob

IBM Bob dipakai sebagai AI coding partner untuk membangun antarmuka web (`index.html`): tab chat, rekap penjualan, hitung pesanan, dan pengaturan koneksi ke Langflow. Bob juga terhubung ke MCP server Langflow sehingga flow AdminKu dapat dipanggil sebagai tool dari Bob.

### Integrasi Langflow dan Bob

| | Peran |
|---|---|
| **Langflow** | Menjalankan logika AI dan membuka flow lewat REST API dan MCP server |
| **Bob** | Membangun antarmuka web yang memanggil flow, dan terhubung ke MCP Langflow |
| **Aliran data** | Pesan pemilik → endpoint API Langflow → Agent + tool → teks jawaban → ditampilkan di web |
| **Output akhir** | Aplikasi web AdminKu dengan draf balasan, caption, rekap penjualan, dan penghitung pesanan |

## Struktur Repositori

```
.
├── AdminKu_-_Asisten_Toko_UMKM.json                  # Flow Langflow (import ke Langflow)
├── index.html                                        # Antarmuka web AdminKu
├── knowledge_dapur_bu_rina.txt                       # Katalog dan FAQ toko (data contoh)
├── Data_Penjualan_Dapur_Bu_Rina_-_Transaksi.csv      # Data transaksi contoh
├── Data_Penjualan_Dapur_Bu_Rina_-_Ringkasan.csv      # Ringkasan penjualan contoh
└── screenshots/                                      # Tangkapan layar
```

## Cara Menjalankan

### Prasyarat

- IBM Langflow (berjalan lokal atau di server)
- Akun Astra DB (DataStax) untuk vector store
- API key untuk model embedding Google dan LLM yang dipakai Agent
- Akun Composio dengan koneksi Google Sheets
- Google Sheets berisi dua sheet: `Ringkasan` dan `Transaksi` (format mengikuti file CSV contoh)

### Langkah

1. **Import flow.** Di Langflow, import `AdminKu_-_Asisten_Toko_UMKM.json`.
2. **Isi kredensial** pada komponen Astra DB (token dan endpoint database), model embedding, model Agent, dan Google Sheets (koneksi Composio).
3. **Sesuaikan spreadsheet.** Ganti ID spreadsheet di System Prompt Agent dengan ID Google Sheets milikmu.
4. **Muat knowledge.** Pada komponen `Read File`, pilih `knowledge_dapur_bu_rina.txt` (atau data tokomu), lalu jalankan jalur ingest agar data masuk ke Astra DB.
5. **Uji di Playground** Langflow, misalnya: *"Kak, nastar keju berapa harganya dan tahan berapa lama?"*
6. **Buka antarmuka.** Buka `index.html` di browser, klik **Pengaturan Langflow**, lalu isi alamat Langflow, Flow ID (dari URL flow), dan API key Langflow.

Alamat Langflow, Flow ID, dan API key disimpan di browser saja. Pakai hanya di komputer pribadi.

## Data Contoh

Semua data adalah **data contoh fiktif** untuk keperluan demo.

Ringkasan data penjualan contoh (4 minggu, 31 transaksi):

- Total penjualan: Rp10.849.000
- Produk terlaris (unit): Basreng Pedas, 84 unit
- Omzet terbesar: Nastar Keju, Rp3.145.000
- Omzet mingguan (minggu 36 sampai 39): Rp2.029.000, Rp2.307.000, Rp2.710.000, Rp3.803.000

## Dampak yang Diharapkan

Berkurangnya pekerjaan admin manual: membalas pertanyaan pelanggan, menulis caption, dan merekap penjualan ditangani satu asisten. Balasan menjadi lebih konsisten karena bersumber dari katalog dan FAQ yang sama. Besaran penghematan waktu belum diukur. Indikator yang direncanakan untuk uji coba: waktu admin per hari sebelum dan sesudah, waktu balas pertama ke pelanggan, dan kesesuaian jawaban dengan katalog.

## Pengembangan Selanjutnya

- Cek stok dan eskalasi pertanyaan ke pemilik
- Laporan mingguan otomatis
- Integrasi ke kanal chat pelanggan
- Pemisahan data per toko agar satu sistem melayani banyak toko
- Penerapan ke usaha jasa lain seperti katering, salon, dan bengkel

## Keamanan

- Jangan commit API key, token Astra DB, atau kredensial lain ke repositori.
- Periksa file flow `.json` sebelum dipublikasikan agar tidak memuat token atau rahasia lain.
- Agent hanya memiliki akses baca ke Google Sheets.

## Teknologi

IBM Langflow · IBM Bob · Astra DB (DataStax) · Google Generative AI Embeddings · Composio (Google Sheets) · HTML, CSS, JavaScript
