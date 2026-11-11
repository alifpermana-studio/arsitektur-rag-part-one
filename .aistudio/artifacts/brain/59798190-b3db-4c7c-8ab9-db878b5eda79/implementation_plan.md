# Rencana Implementasi: Masterclass RAG Bagian 1 — Unified Motion Graphic Player (`Full.html`)

## 1. Gambaran Umum & Keputusan Desain
Aplikasi ini menggabungkan seluruh 9 *cutscene* motion graphic **Masterclass RAG Bagian 1** (`Scene 1` s.d. `Scene 9`) beserta animasi penutup **Alif Pustaka Logo Reveal** (`Outro`) menjadi satu aplikasi pemutar dan perekam video **Vanilla HTML5 (`<canvas>` 2D + Web Audio API)** yang utuh dan mandiri tanpa *framework* eksternal.

Berdasarkan konfirmasi kebutuhan:
1. **Pratinjau Langsung & File Mandiri**: Menghasilkan file mandiri untuk produksi sekaligus menyajikannya sebagai halaman utama agar langsung tampil di layar *preview* port 3000.
2. **Pemuatan Audio Voiceover Hibrida**: Memuat 9 file audio narasi lokal (`WAV`) secara asinkron melalui Web Audio API dengan *fallback* dekoder Base64/PCM bawaan sehingga pemutaran dan perekaman tetap berjalan stabil di berbagai lingkungan.
3. **Timeline Master Terpadu**: Menggabungkan ke-9 scene dan Outro dalam satu *timeline* berurutan yang dilengkapi tombol lompat instan per-scene (`Scene 1`..`Scene 9` + `Outro`), kontrol `Play/Pause`, `Reset`, *seekbar*, *toggle* `Narasi`/`Musik`/`Efek`/`Subtitle`, unduh `.srt` gabungan, serta ekspor video `MP4` (`Mediabunny` WebCodecs + *fallback* `MediaRecorder`).

---

## 2. Arsitektur Sistem & Alur Timeline

```
+-----------------------------------------------------------------------------------+
|                        MASTER PLAYER & TIMELINE CONTROLLER                        |
|  [Play/Pause] [Reset] [Seekbar 0:00..Total] [Scene 1..9 + Outro] [MP4] [SRT]      |
+-----------------------------------------+-----------------------------------------+
                                          |
                   masterTime (0 .. TOTAL_DUR) -> { sceneIdx, localT }
                                          |
        +---------------------------------+---------------------------------+
        |                                                                   |
        v                                                                   v
+-------------------------------+                                   +---------------+
|   CANVAS 2D RENDER ENGINE     |                                   | WEB AUDIO API |
|   (1080x1920 @ 30 FPS, 9:16)  |                                   | MIXER ENGINE  |
+-------------------------------+                                   +---------------+
| 1. drawThemeBg(g)             |                                   | - Voiceover   |
|    - Latar #0e1626            |                                   |   Buffer 1..9 |
|    - Lingkaran glow #13203a   |                                   | - Procedural  |
| 2. Namespace Renderer:        |                                   |   Ambient BGM |
|    - renderScene1..9(g, t, D) |                                   | - Per-Scene   |
|      * tClamped = min(t, VO_D)|                                   |   Synth SFX   |
|      * Animasi ambient jalan  |                                   | - Outro SFX   |
|      * Simetri x = 540        |                                   +---------------+
|    - renderOutro(g, t, 6.0)   |
| 3. drawSubtitleOverlay(g)     |
|    - Kapsul #000000 (a=0.68)  |
|    - Cue terakhir +1.8 dtk    |
+-------------------------------+
```

### Perhitungan Durasi & Jeda Baca 2 Detik (`SCENE_DELAY = 2.0`)
Setiap scene memiliki durasi total `DURS[i] = VO_DURS[i] + 2.0`:
- **Rentang Aktif Voiceover (`0 <= localT < VO_DURS[i]`)**: Seluruh animasi struktural, transisi sub-scene, dan narasi berjalan normal sesuai *timestamp* asli.
- **Rentang Jeda Baca (`VO_DURS[i] <= localT < VO_DURS[i] + 2.0`)**:
  - Waktu struktural ditahan pada akhir voiceover (`tClamped = Math.min(localT, VO_DURS[i] - 0.001)`) agar diagram dan teks kesimpulan di sub-scene terakhir tetap utuh di layar dan nyaman dibaca.
  - Waktu *ambient* (`localT`) tetap berjalan untuk animasi partikel mengalir, gerakan napas/mengangguk karakter, kilau sinyal, serta musik latar.
  - *Cue* subtitle terakhir pada setiap scene diperpanjang `+1.8 detik` memasuki masa jeda baca.

---

## 3. Standarisasi Tema Warna (Dark Tech Theme)

Seluruh scene dan pembungkus halaman diseragamkan menggunakan palet `K`:

| Token | Hex | Peruntukan Visual |
| :--- | :--- | :--- |
| `K.bg` | `#0e1626` | Latar belakang utama kanvas `1080×1920` di semua scene |
| `K.bg2` | `#13203a` | Dua lingkaran dekoratif sudut kanvas di `(980, 160, r=300)` dan `(60, 1800, r=360)` |
| `K.ink` | `#172235` | Isi kartu/panel gelap, gedung, serta teks di atas permukaan terang (`K.iv` / `K.amb`) |
| `K.surf` | `#22314d` | Permukaan sekunder, *border* kartu/subtitle, dan *track* luar *progress bar* |
| `K.iv` | `#F6F5EE` | Teks judul/utama di atas latar gelap `#0e1626` & permukaan kartu terang |
| `K.cob` | `#4667FF` | Aksen utama biru teknologi, tombol utama, pipa Query |
| `K.teal` | `#16B99A` | Aksen sukses, sumber terverifikasi, tombol scene aktif (`.act`) |
| `K.cor` | `#FF765E` | Aksen peringatan, atap, sorotan penting |
| `K.amb` | `#FFC65C` | Aksen *badge* nomor, kilau kunci, partikel aktif |
| `K.peri` | `#A3A5F4` | Teks subjudul, aksen ungu lembut, konektor sekunder |
| `K.sl` | `#73819A` | Elemen redup, label sekunder, garis bantu |
| `K.red` | `#E5484D` | Indikator bahaya, *hallucination*, *OOMKilled*, error |
| `K.wh` | `#FFFFFF` | Sorotan putih murni |

**Aturan Kontras Teks & Latar:**
- Semua latar krem/terang (`#F6F5EE`, `#E8E7DC`, `#A3A5F4`) pada sub-scene asli diganti dengan `drawThemeBg(g)` (`#0e1626` + 2 lingkaran `#13203a`).
- Semua teks judul/utama dan garis *outline* utama yang semula berwarna gelap (`#172235` / `#001531`) karena berada di atas latar terang diubah menjadi `K.iv` (`#F6F5EE`) agar kontras tajam di atas `#0e1626`.
- Teks atau ikon yang berada di dalam kartu/kapsul berwarna terang (`K.iv` `#F6F5EE` atau `K.amb` `#FFC65C`) tetap dipertahankan berwarna gelap `K.ink` (`#172235`).

---

## 4. Preservasi Animasi Bawaan & Perbaikan Simetri Per-Scene

Seluruh fungsi gambar, karakter, diagram, *keyframe*, dan kurva *easing* asli dari kesembilan scene dipertahankan 100% di dalam *namespace* terisolasi (`renderScene1` s.d. `renderScene9`), dengan perbaikan simetri terhadap sumbu tengah vertikal kanvas (`x = 540`):

1. **Scene 1 — Mengenal RAG (Retrieval-Augmented Generation)**
   - **Preservasi**: Robot LLM dengan kacamata baca, gelembung pikir `?`, gedung perpustakaan dokumen, jembatan cahaya 3 tahap (*Ambil*, *Gabung*, *Jawab*), dan kartu kesimpulan RAG.
   - **Perbaikan Simetri & Tema**: Menyelaraskan judul utama dan robot tepat di sumbu `x = 540`, menyeimbangkan posisi robot dan gedung saat berdampingan (`x = 270` dan `x = 810`), menjaga jembatan partikel simetris di antara keduanya, serta memusatkan transisi *radial wipe* di `(540, 960)`.

2. **Scene 2 — Analogi Buku Terbuka vs Ujian Tutup Buku**
   - **Preservasi**: Papan jadwal sekolah, karakter murid *Open-Book* vs *Closed-Book*, rak buku referensi dengan lup pencari, buku catatan terbuka, dan kartu kutipan sitasi.
   - **Perbaikan Simetri & Tema**: Merapikan pasangan kartu perbandingan *Tutup Buku* dan *Buka Buku* agar berjarak seimbang dari `x = 540`, merapikan garis konektor dari rak referensi menuju halaman buku, dan mengubah teks judul gelap di latar terbuka menjadi `K.iv`.

3. **Scene 3 — Tiga Kelemahan Fatal LLM Murni**
   - **Preservasi**: Kalender terkunci (*Knowledge Cutoff*), gelembung percakapan halusinasi dengan meteran *Yakin 99% / Akurat 0%*, brankas dokumen internal terkunci, 3 kartu ringkasan masalah, dan kapsul solusi RAG.
   - **Perbaikan Simetri & Tema**: Menjaga *progress bar* meteran *Yakin 99%* dan *Akurat 0%* tetap **rata kiri** di dalam *track* yang terpusat di tengah kartu, menyeimbangkan ke-3 kartu masalah secara vertikal di `x = 540`, serta memusatkan transisi *radial wipe* di `(540, 960)`.

4. **Scene 4 — Ilusi AI Membaca File & Cara Kerja Tool Calling**
   - **Preservasi**: Animasi kartu PDF masuk ke kotak chat, robot memilih 3 *tool* (`search_docs`, `query_db`, `python_exec`), *timeline* urutan JSON *Tool Call* 3 langkah, dan penggabungan hasil ke kalimat jawaban.
   - **Perbaikan Simetri & Tema**: Memperbaiki pusat grup 3 kartu *tools* (semula `x = 110 + i*295` yang berat ke kiri menjadi terpusat simetris di `x = 540`), menyeimbangkan posisi robot dan panel dokumen di sub-scene 1, serta menyeragamkan pusat *radial wipe* ke `(540, 960)`.

5. **Scene 5 — Eksperimen Membongkar System Prompt & JSON Schema Tool**
   - **Preservasi**: Kartu *Prompt Pengujian*, kaca pembesar memindai cetak biru *System Prompt*, kartu skema JSON `search_knowledge_base`, aliran sinyal menuju *Backend Application*, dan 3 kartu fakta bahwa *Tool Calling* bukan RAG penuh.
   - **Perbaikan Simetri & Tema**: Menyelaraskan posisi kartu *System Prompt*, skema JSON, dan *Backend* tepat pada sumbu tengah `x = 540`, merapikan kabel konektor dan partikel agar berangkat/tiba tepat di tengah *port* kartu, serta memusatkan *radial wipe* di `(540, 960)`.

6. **Scene 6 — Tiga Skenario Saat Pengguna Mengunggah Dokumen ke AI**
   - **Preservasi**: Dokumen bercabang ke 3 jalur (`1`..`3`), gerbong kereta *Context Stuffing* 200k token, pisau pemotong *Quick Chunking*, mesin *Vector DB* & *Hybrid Search* dengan stempel *VERIFIED*, dan matriks perbandingan 3 kolom.
   - **Perbaikan Simetri & Tema**: Menyeimbangkan cabang panah 3 skenario terhadap `x = 540`, menjaga bar kapasitas *Context Window* tetap mengisi **rata kiri** di dalam *track* yang simetris, serta mengubah teks gelap di atas latar terbuka menjadi `K.iv`.

7. **Scene 7 — Arsitektur Lengkap RAG: Ingestion Pipeline vs Query Pipeline**
   - **Preservasi**: Dua pabrik bertingkat (*Jalur 1 Persiapan Data* & *Jalur 2 Menjawab Pertanyaan*), ban berjalan dokumen → *chunking* → *embedding* `[0.12, -0.84, 0.53]` → silinder *Vector DB*, dan aliran *Query* pengguna → *Retrieval* → *LLM* → Jawaban.
   - **Perbaikan Simetri & Tema**: Menyelaraskan posisi 4 stasiun proses pada Jalur 1 dan Jalur 2 agar simetris terhadap `x = 540`, merapikan pipa konektor vertikal/horizontal beserta aliran partikelnya, dan memusatkan transisi *wipe* di `(540, 960)`.

8. **Scene 8 — Bencana Naive RAG di Dunia Nyata**
   - **Preservasi**: Ekspektasi 3 langkah mudah vs 6 kartu kegagalan produksi, gergaji mesin *Fixed-Size Chunking* membelah tabel laporan keuangan, kabut pencarian vektor tersesat (*Vector Search*), balon konteks membengkak (*Lost in the Middle*), robot halusinasi, dan perisai *Production-Grade RAG*.
   - **Perbaikan Simetri & Tema**: Menyeimbangkan tata letak *grid* kartu masalah (2 kolom × 3 baris) dan 3 pilar perisai *Production RAG* tepat di sumbu `x = 540`, serta memastikan semua titik pusat transisi lingkaran berada di `(540, 960)`.

9. **Scene 9 — Cetak Biru Mikroservis Production RAG & Tantangan VPS 4 GB**
   - **Preservasi**: Diagram 2 pipa paralel (*Ingestion* & *Query*), 3 kontainer layanan (*Next.js*, *Express.js*, *Redis*), terminal `$ docker ps` 5 kontainer (`5/5 Up`), mesin VPS 4 GB RAM / 2 vCPU, blok model *embedding* lokal `2,2 GB` yang memicu `OOMKilled`, dan kartu *teaser* Part 2 (*Docker Compose*).
   - **Perbaikan Simetri & Tema**:
     - Memperbaiki *header* di setiap sub-scene agar terpusat dan menggunakan `K.iv` untuk judul utama serta `K.peri` untuk subjudul.
     - Memperbaiki posisi 3 kontainer di sub-scene 2 (`Next.js`, `Express.js`, `Redis`) yang semula miring/zig-zag (`x = 90, 190, 140` dengan lebar `780`) menjadi rata tengah simetris di `x = 150` (`540 - 780/2`) dengan garis konektor vertikal tepat di `x = 540`.
     - Menyelaraskan pipa dan 4 *node* di sub-scene 1 (`x = 165, 415, 665, 915`) agar simetris terhadap `x = 540`.
     - Menyelaraskan bar kapasitas RAM 4 GB di sub-scene 5 (`x = 90, w = 900`, rata kiri untuk segmen terpakai) dan grup panel *Docker Compose* + bar RAM + termometer di sub-scene 6 agar seimbang kiri-kanan terhadap `x = 540`.
     - Mengganti titik pusat *wipe* `ORG` ke `(540, 960)`.

10. **Outro — Alif Pustaka Logo Reveal (`6.0 detik`)**
    - **Preservasi & Integrasi**: Menampilkan ke-10 *path* vektor asli logo Alif Pustaka dengan warna aslinya dan ke-11 *path* *wordmark* `"alifpustaka"` berwarna `K.iv` (`#F6F5EE`), partikel *ambient* & *burst*, cincin gelombang, *typewriter* `"Ruang Literasi, Rumah Inovasi"`, serta kapsul URL `https://alifpustaka.web.id` di atas `drawThemeBg`.

---

## 5. Verifikasi & Kriteria Selesai
- Memastikan seluruh 9 scene dan Outro dapat diputar berurutan maupun dilompati lewat tombol scene tanpa *error* konsol atau bentrok variabel.
- Memastikan *subtitle* tampil di kotak kapsul bawah yang seragam di semua scene dan diperpanjang `+1.8 detik` pada *cue* terakhir tiap scene.
- Memastikan fitur unduh `.srt` dan `Download Video (MP4)` berfungsi penuh dari awal hingga akhir *timeline*.
