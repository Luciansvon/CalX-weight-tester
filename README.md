# CalX

**FEA & LOAD TESTING** — Perangkat lunak desktop mandiri lintas-CAD untuk simulasi uji beban struktural (*Finite Element Analysis* / FEA) dan validasi desain produk maupun furnitur dengan mesin inti **Rust**.

* **Status Proyek:** Fase 0 — Pondasi & Dokumentasi
* **Lini Masa & Target Fase:** [TIMELINE.md](TIMELINE.md)
* **Patokan Tampilan V1.0:** [docs/pratinjau-tampilan-v1.html](docs/pratinjau-tampilan-v1.html)

---

## 1. Konsep & Strategi Integrasi

**CalX** dirancang sebagai ruang uji beban digital yang berdiri terpisah di luar aplikasi gambar 3D agar terbebas dari batasan performa Ruby API SketchUp, memiliki kendali penuh atas rendering GPU, tidak membuat aplikasi CAD mengalami *crash*, serta mampu menguji rancangan dari berbagai perangkat lunak 3D (SketchUp, AutoCAD, Autodesk Inventor, SolidWorks, Fusion 360, FreeCAD, dan Blender).

* **Strategi Jembatan SketchUp (`CalX Link` / `CalX Bridge`):** Plugin jembatan ringan (±50–80 baris skrip Ruby) berisi satu tombol di dalam SketchUp yang otomatis mengekspor model ke format `.STL` atau `.OBJ` lalu langsung membuka aplikasi **CalX**.
* **Ekosistem Penamaan:**
  * Format berkas proyek: `.calx`
  * Workspace & crate Rust: `calx-core`, `calx-cli`, `calx-ui` (dengan opsi nama publikasi `calx-fea` agar tidak bentrok dengan crate lama).
  * Opsi tagline: *"Universal Structural Load Testing"*, *"Test Before You Build"*, *"Uji Beban Universal, Lintas CAD"*.
* **Identitas Visual:**
  * **Ikon:** Huruf **X** geometris minimalis dengan sentuhan *bevel* bersih dan gradasi peta warna FEA (biru dingin di sudut luar menuju oranye-merah menyala di pusat silang).
  * **Tipografi:** Huruf sans-serif modern yang kokoh — teks `Cal` berwarna kontras (putih pada tema gelap, gelap pada tema terang) dan `X` bergradasi oranye-merah, dilengkapi subjudul `FEA & LOAD TESTING`.

---

## 2. Dukungan Format Berkas Universal

**CalX** memfokuskan jalur input pada dua standar utama agar ramah bagi pengguna pemodelan cepat sekaligus presisi bagi pengguna CAD teknik:

1. **Jalur CAD Presisi (`.STEP` / `.stp`):**
   * Standar pertukaran data dari Inventor, SolidWorks, Fusion 360, Rhino, AutoCAD, dan FreeCAD.
   * Menyimpan geometri *B-Rep / Solid* asli sehingga bidang datar, lubang, dan silinder langsung dikenali sebagai satu permukaan (*face*) utuh.
2. **Jalur Mesh Universal (`.STL`, `.OBJ`, `.3MF`):**
   * Mudah diekspor langsung dari SketchUp maupun Blender.
   * Dilengkapi dua pemrosesan wajib di backend:
     * **Pemeriksaan Watertight & Perbaikan Otomatis:** Memastikan model berstatus *solid* tertutup tanpa lubang (*non-manifold*) sebelum masuk proses *meshing*.
     * **Smart Face Selection (berdasarkan sudut):** Pengelompokan segitiga bersebelahan secara otomatis memakai ambang batas sudut normal (`< 15°`) agar pengguna cukup 1 klik untuk memilih seluruh permukaan datar atau sisi lengkung.

---

## 3. Arsitektur Teknis & Strategi Evolusi Tampilan

Seluruh logika simulasi diisolasi ke dalam pustaka inti Rust (`calx-core`) tanpa ketergantungan pada kerangka antarmuka tertentu, terpisah dari utilitas pengujian terminal (`calx-cli`) dan antarmuka desktop (`calx-ui`).

### Mesin Simulasi (Sidecar Open-Source)
* **Volume Mesher — Gmsh:** Mengubah berkas `.STEP` dan `.STL` bersih menjadi jaring tetrahedral 3D (*volume mesh*). Opsional: **fTetWild** untuk menangani berkas `.STL` dari SketchUp yang geometrinya kurang rapi atau saling tumpang tindih.
* **FEA Solver — CalculiX (`ccx`):** Solver ringan yang dapat dibundel langsung di dalam folder aplikasi, mendukung pemrosesan multi-inti melalui `OMP_NUM_THREADS`, material isotropik & ortotropik, kontak multi-komponen, uji statis, uji tekuk (*buckling*), hingga uji getaran (*modal*).

### Keputusan Tampilan Tahap Awal & Evolusi
* **Keputusan Tahap Awal (Pilihan A):** **Tauri v2 + antarmuka web modern + kanvas 3D Three.js**. Pendekatan ini dipilih agar aplikasi cepat matang, tata letak modern nyaman dipakai, kendali kamera (*OrbitControls*), pemilihan bidang (*raycasting*), dan rendering *heatmap* via `BufferGeometry` siap digunakan—dengan transfer array biner `Float32Array` dari Rust melalui IPC.
* **Kesiapan Opsi Native (Pilihan B):** Karena seluruh logika fisika dan data terpusat pada `calx-core`, di masa depan proyek tetap leluasa menambahkan atau beralih ke antarmuka murni Rust (**`egui` + `wgpu` / `three-d`**) tanpa membongkar ulang mesin perhitungannya.

### Alur Data `calx-core` (Pipeline 5 Tahap)
1. **Import & Validasi:** Membaca `.STL`/`.OBJ` di Rust (atau meneruskan `.STEP` ke `gmsh`), memverifikasi status *watertight*, dan menormalisasi satuan.
2. **Smart Face Grouping:** Algoritma *flood-fill* (BFS) mengelompokkan segitiga berdasarkan kemiripan vektor normal menjadi *Logical Surface ID*.
3. **Volume Meshing & Node Mapping:** Menjalankan `gmsh` di *background thread*, lalu menggunakan struktur spasial *KD-Tree* untuk memetakan area *Fixed Support* dan *Force* ke himpunan node (`*NSET`).
4. **Generator Deck `.inp` & Eksekusi Solver:** Menulis berkas `.inp` (`*NODE`, `*ELEMENT`, `*MATERIAL`, `*BOUNDARY`, `*CLOAD`/`*DLOAD`) dan menjalankan `ccx` secara asinkron sambil memantau log `stdout`.
5. **Post-Processing & Ekstraksi Kulit Luar:** Membaca hasil `.frd`, menghitung perpindahan `(dx, dy, dz)` dan *Von Mises Stress* secara paralel dengan `rayon`, lalu mengekstrak permukaan kulit luar (*exterior faces*) dalam format `Vec<f32>` untuk rendering 60 FPS di viewport.

### Rencana Struktur Workspace & Crate Rust
```text
calx/
├── binaries/                # Sidecar terbundel (gmsh, ccx)
├── crates/
│   ├── calx-core/           # Mesin inti pure Rust (tanpa dependensi UI)
│   │   └── src/
│   │       ├── importer/    # Parser STL/OBJ + validasi watertight
│   │       ├── geometry/    # Smart Face Grouping & KD-Tree node mapping
│   │       ├── mesher/      # Wrapper CLI Gmsh & parser .msh
│   │       ├── solver/      # Generator deck .inp & runner CLI CalculiX
│   │       ├── post/        # Parser .frd, rumus Von Mises, skin extractor
│   │       └── materials/   # Database preset material (logam, kayu, polimer)
│   ├── calx-cli/            # Prototipe CLI untuk pengujian cepat
│   └── calx-ui/             # Aplikasi desktop (Tauri v2; opsi masa depan egui/wgpu)
└── Cargo.toml
```

* **Crate Utama `calx-core`:** `stl_io`, `tobj` (parser mesh awal), `glam` (matematika 3D & SIMD), `kiddo` atau `rstar` (spatial lookup), `rayon` & `tokio` (paralelisme & asinkron), `serde` & `bytemuck` (serialisasi & buffer *zero-copy*), serta `notify` (pengawas berkas *live-link*).

---

## 4. Patokan Tampilan Antarmuka V1.0

Patokan visual V1.0 tersimpan di [docs/pratinjau-tampilan-v1.html](docs/pratinjau-tampilan-v1.html) dengan prinsip desain berikut:
* **Tema Ganda Terkalibrasi:** **Gelap Presisi (`#0B0D10`)** dan **Terang Bersih (`#F2F4F7`)**, panel kaca buram halus, serta satu aksen utama oranye-merah (`#F04E23`) yang selaras dengan huruf `X` pada logo.
* **Proporsi Ruang Kerja:** ~75% kanvas pratinjau 3D penuh di kiri dan ~25% panel kendali mengambang di kanan.
* **Alur Ramah Desainer:** Masukan beban menggunakan kilogram (`kg`) dengan konversi otomatis ke Newton (`N`), lencana status tegas **AMAN (LOLOS)** / **RAWAN (GAGAL)**, serta 4 kartu metrik utama (Faktor Keamanan, Lendutan Maksimal, Tegangan Tertinggi, dan Perkiraan Berat).

### Penyempurnaan Wajib Saat Implementasi V1.0 Asli
1. **Pemilih Bidang Tumpuan & Beban:** Tombol dan interaksi klik permukaan 3D untuk menentukan area **Tumpuan Tetap** dan **Area Beban** pada berkas CAD apa pun.
2. **Pin Ukur Virtual & Skala Lenturan:** Fitur menempelkan pin bacaan tegangan (`MPa`) dan lendutan (`mm`) di titik tertentu serta penggeser pembesar visual lenturan.
3. **Navigasi Kamera & Progres Hitung:** Tombol arah pandang cepat (Depan, Samping, Atas, Isometrik, Reset) serta penanda progres selama `gmsh` dan `ccx` berjalan.
4. **Skala Warna Otomatis:** Batang legenda tegangan vertikal (`0 MPa` hingga tegangan puncak) menyesuaikan rentang kekuatan material yang sedang diuji secara otomatis.

---

## 5. Tiga Aturan Wajib Akurasi FEA

1. **Konsistensi Sistem Satuan (`mm - Tonne - s - N - MPa`):**
   * Koordinat dan panjang model: `mm`
   * Gaya beban: `N` (contoh: massa `100 kg ≈ 981 N`)
   * Modulus Elastisitas ($E$) dan *Yield Strength*: `MPa` ($\text{N/mm}^2$)
   * Keluaran otomatis: Lendutan dalam `mm` dan tegangan dalam `MPa`.
2. **Gunakan Elemen Orde 2 (`C3D10`):**
   * Wajib menjalankan Gmsh dengan flag `-order 2` untuk menghasilkan elemen tetrahedral 10-titik (`C3D10`), mencegah efek *shear locking* (kekakuan semu berlebih pada simulasi lentur) yang muncul pada elemen `C3D4`.
3. **Perhitungan Tegangan Ekuivalen Von Mises:**
   * Dari 6 komponen tensor tegangan tiap node ($\sigma_{xx}, \sigma_{yy}, \sigma_{zz}, \tau_{xy}, \tau_{yz}, \tau_{zx}$) keluaran CalculiX, tegangan ekuivalen dihitung paralel di Rust menggunakan persamaan:

$$\sigma_v = \sqrt{\frac{1}{2}\left[(\sigma_{xx}-\sigma_{yy})^2 + (\sigma_{yy}-\sigma_{zz})^2 + (\sigma_{zz}-\sigma_{xx})^2 + 6(\tau_{xy}^2 + \tau_{yz}^2 + \tau_{zx}^2)\right]}$$

---

## 6. Peta Jalan Singkat & Infrastruktur Bersama

| Versi | Fokus Utama | Tujuan |
|---|---|---|
| **V1.0 (Inti)** | Uji Beban Statis, Preset Material, Estimasi Berat, Virtual Probe, CalX Link | Pondasi mesin hitung dan aplikasi inti yang langsung siap dipakai validasi nyata. |
| **V1.5 (Pembeda)** | Uji Anti-Jungkang (CoG), Arah Serat Ortotropik, Live-Link Watcher, Auto-Report PDF/DOCX | Fitur pembeda untuk desainer produk, furnitur, dan komponen cetak 3D. |
| **V2.0 (Lanjutan)** | Uji Tekuk, Uji Getaran, Evaluator Sambungan, Mode Perbandingan Side-by-Side | Validasi desain menyeluruh dari tahap purwarupa hingga siap produksi. |

Rincian daftar kerja dan kriteria selesai setiap fase tersedia di [TIMELINE.md](TIMELINE.md).

Untuk menjaga konsistensi kualitas lintas proyek, otomatisasi pengujian, benchmark, audit keamanan, dan validasi rilis nantinya akan dihubungkan ke infrastruktur bersama `Luciansvon/B.I.M.A-DEV-INFRA` tanpa menyalin atau menduplikasi *reusable workflow* ke dalam repositori ini.
