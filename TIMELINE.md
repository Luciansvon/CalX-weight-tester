# Lini Masa Pengembangan CalX

Dokumen ini memetakan tahapan pembangunan **CalX** secara berurutan dari pondasi dokumentasi hingga fitur validasi produksi lanjutan, beserta rincian tugas dan kriteria selesai yang dapat diverifikasi pada setiap fase.

---

## Ringkasan Fase & Status

| Fase | Target Versi | Fokus Utama | Status |
|---|---|---|---|
| **Fase 0** | Pondasi | Konsep final, patokan tampilan V1.0, repositori GitHub, dan dokumentasi awal | Sedang Diselesaikan |
| **Fase 1** | V1.0 (Bagian 1) | Kerangka Rust workspace, Tauri v2 + Three.js, import `.STL`/`.OBJ`, dan pemilihan bidang | Direncanakan |
| **Fase 2** | V1.0 (Bagian 2) | Sidecar Gmsh (`C3D10`) & CalculiX (`ccx`), preset material, peta warna dinamis, Virtual Probe | Direncanakan |
| **Fase 3** | Rilis V1.0 | Jembatan SketchUp (`CalX Link`), dukungan `.STEP`, uji akurasi, audit DEV-INFRA, pemaketan | Direncanakan |
| **Fase 4** | V1.5 (Pembeda) | Anti-jungkang (CoG), arah serat ortotropik, Mesh Doctor, Live-Link, preset standar, laporan PDF/DOCX | Direncanakan |
| **Fase 5** | V2.0 (Lanjutan) | Uji tekuk, uji getaran, evaluator sambungan, zona hemat bahan, irisan 3D, opsi native Pilihan B | Direncanakan |

---

## Rincian Setiap Fase

### Fase 0 — Pondasi Proyek, Patokan Tampilan & Dokumentasi
**Tujuan:** Mengunci arah produk, standar akurasi fisika, dan patokan antarmuka V1.0 sebelum penulisan kode aplikasi dimulai.

* [x] Penetapan konsep produk mandiri lintas-CAD dan strategi integrasi jembatan SketchUp (`CalX Link`)
* [x] Penguncian patokan tampilan antarmuka V1.0 pada [docs/pratinjau-tampilan-v1.html](docs/pratinjau-tampilan-v1.html) (tema Gelap Presisi & Terang Bersih, komposisi 75/25)
* [x] Penyiapan repositori GitHub `Luciansvon/CalX-weight-tester` pada cabang `main`
* [x] Penyusunan [README.md](README.md), `TIMELINE.md`, dan `.gitignore`

**Kriteria Selesai:** Seluruh berkas pondasi tersimpan rapi di cabang `main` repositori GitHub dan siap dijadikan acuan eksekusi Fase 1.

---

### Fase 1 — Kerangka Aplikasi & Kanvas 3D Nyata (V1.0 Bagian 1)
**Tujuan:** Membangun kerangka proyek modular (`calx-core`, `calx-cli`, `calx-ui`) dan viewport 3D interaktif yang setia pada patokan tampilan V1.0.

* [ ] Inisialisasi Cargo workspace (`crates/calx-core`, `crates/calx-cli`, `crates/calx-ui`) dan proyek **Tauri v2**
* [ ] Implementasi antarmuka bertema ganda (Gelap Presisi & Terang Bersih) serta bilah atas satu baris (logo CalX, tombol Impor Model, status `CalX Link`, pengalih tema)
* [ ] Pembangunan modul `importer/` untuk membaca berkas `.STL` dan `.OBJ` beserta pemeriksaan *watertight* (*non-manifold*)
* [ ] Pembangunan modul `geometry/` untuk *Smart Face Grouping* menggunakan BFS dengan ambang sudut normal (`< 15°`)
* [ ] Pembangunan kanvas 3D `Three.js` dengan kendali orbit, tombol arah pandang cepat (Depan, Samping, Atas, Isometrik, Reset), dan penanda satuan aktif (`mm · N · MPa`)
* [ ] Interaksi pemilihan permukaan 3D untuk menandai bidang **Tumpuan Tetap** dan **Area Beban** dari panel kanan

**Kriteria Selesai:** Pengguna dapat membuka berkas `.STL`/`.OBJ`, memutar model di kanvas 3D secara mulus, serta mengeklik permukaan untuk menandai area Tumpuan Tetap dan Area Beban.

---

### Fase 2 — Mesin Hitung Fisika & Peta Warna Otomatis (V1.0 Bagian 2)
**Tujuan:** Menghubungkan antarmuka dengan mesin simulasi FEA sungguhan (Gmsh + CalculiX) dan menampilkan hasil tegangan serta lendutan akurat.

* [ ] Pengemasan sidecar **Gmsh** (wajib `-order 2` menghasilkan elemen `C3D10`) dan **CalculiX (`ccx`)** di folder `binaries/`
* [ ] Pemetaan area Tumpuan & Beban dari permukaan terpilih ke himpunan node (`*NSET`) menggunakan *KD-Tree* (`kiddo` atau `rstar`)
* [ ] Pembangunan modul `solver/`: generator berkas `.inp` dengan sistem satuan konsisten `mm - Tonne - s - N - MPa` dan eksekutor asinkron `ccx` + pembaca progres `stdout`
* [ ] Pembangunan modul `post/`: pembaca hasil `.frd`, perhitungan paralel *Von Mises Stress* dan vektor perpindahan dengan `rayon`, serta ekstraksi kulit luar (*exterior faces*) berformat `Float32Array`
* [ ] Pembangunan modul `materials/`: database preset bahan (Kayu Jati, Kayu Lapis/Multiplek, Baja Ringan/Besi Kotak, Aluminium 6061, Plastik PLA) lengkap dengan perhitungan perkiraan berat
* [ ] Peta warna tegangan dengan skala legenda vertikal dinamis, penggeser **Skala Lenturan**, lencana status **AMAN (LOLOS)** / **RAWAN (GAGAL)**, dan fitur **Pin Ukur Virtual** di titik klik model

**Kriteria Selesai:** Menjalankan uji beban pada model 3D menghasilkan nilai tegangan puncak (`MPa`), lendutan maksimal (`mm`), faktor keamanan, perkiraan berat (`kg`), dan peta warna 3D nyata yang dapat diperiksa via CLI maupun desktop UI.

---

### Fase 3 — Jembatan SketchUp (`CalX Link`) & Jalur `.STEP` (Penutup & Rilis V1.0)
**Tujuan:** Menyelesaikan integrasi satu klik dari SketchUp, membuka dukungan berkas CAD teknik `.STEP`, serta memverifikasi akurasi sebelum rilis V1.0.

* [ ] Pembuatan skrip Ruby ringan **CalX Link** (±50–80 baris) dengan 1 tombol di SketchUp untuk mengekspor model ke `.STL`/`.OBJ` dan langsung meluncurkan CalX
* [ ] Dukungan jalur CAD presisi (`.STEP` / `.stp`) diteruskan ke Gmsh beserta pengenalan bidang padat asli
* [ ] Pengujian verifikasi akurasi terhadap kasus uji analitik sederhana (misalnya balok kantilever) dan model furnitur nyata
* [ ] Penghubungan alur pengujian, benchmark, dan audit ke infrastruktur bersama `Luciansvon/B.I.M.A-DEV-INFRA`
* [ ] Penyusunan paket instalasi desktop V1.0 untuk Windows beserta catatan rilis

**Kriteria Selesai:** Alur ujung-ke-ujung dari klik tombol di SketchUp (atau impor `.STEP`) hingga keluar angka dan laporan visual **AMAN / RAWAN** berjalan tanpa galat dan terverifikasi.

---

### Fase 4 — Fitur Pembeda Desain Produk & Furnitur (V1.5)
**Tujuan:** Menambahkan kemampuan khusus yang sangat membantu perancang furnitur, produk kayu/komposit, dan pembuat purwarupa cetak 3D.

* [ ] **Center of Gravity (CoG) & Uji Anti-Jungkang:** Perhitungan titik berat dan stabilitas guling murni di Rust
* [ ] **Arah Serat Material (Ortotropik):** Dukungan sifat material searah/lintang serat kayu maupun lapisan FDM lengkap dengan panah visual 3D arah serat per komponen
* [ ] **Mesh Doctor & Cek Ketebalan Dinding:** Perbaikan lubang mesh otomatis (*auto-cap*) dan penandaan dinding terlalu tipis untuk berkas SketchUp/Blender
* [ ] **Live-Link / Auto-Reload:** Pengawas berkas (`notify`) agar hasil simulasi otomatis diperbarui saat model diedit ulang di CAD
* [ ] **Preset Skenario Uji 1-Klik:** Template beban siap pakai merujuk standar uji produk (misalnya kursi, rak, atau beban gantung)
* [ ] **Generator Laporan Otomatis (PDF / DOCX):** Ekspor laporan uji kelayakan lengkap dengan tangkapan layar *heatmap*, parameter bahan, dan status **PASS / FAIL**

**Kriteria Selesai:** Seluruh fitur pembeda V1.5 teruji pada studi kasus furnitur/produk serta mampu mengekspor berkas laporan PDF/DOCX yang rapi.

---

### Fase 5 — Validasi Produksi Lanjutan & Opsi Native Ringkas (V2.0)
**Tujuan:** Melengkapi alat analisis teknik lanjutan dari tahap purwarupa menuju kesiapan produksi, sekaligus membuka jalur bodi antarmuka murni Rust.

* [ ] **Uji Tekuk (*Buckling Analysis*):** Dukungan mode `*BUCKLE` di CalculiX untuk struktur ramping atau pipa panjang
* [ ] **Uji Getaran (*Modal Analysis*):** Dukungan mode `*FREQUENCY` untuk memeriksa kekakuan dinamis dan frekuensi alami
* [ ] **Evaluator Sambungan (*Joint & Fastener Check*):** Penilaian gaya tarik dan geser pada pertemuan komponen (sekrup, baut, pasak/dowel, purus/tenon, las) tanpa pemodelan ulir detail
* [ ] **Peta Zona Hemat Bahan (*Over-Engineered Indicator*):** Penyorotan area bertegangan sangat rendah yang aman ditipiskan untuk efisiensi bobot dan biaya
* [ ] **Irisan 3D (*Clipping Plane*) & Mode Perbandingan (*Side-by-Side*):** Pemotongan model interaktif dan tabel pembanding variasi material/desain
* [ ] **Jalur Opsional Antarmuka Native Ringkas (Pilihan B):** Pembangunan *frontend* murni Rust (`egui` + `wgpu` / `three-d`) di atas `calx-core` yang sudah matang

**Kriteria Selesai:** Semua modul analisis lanjutan berjalan terintegrasi di atas `calx-core` dengan hasil uji yang konsisten dan tervalidasi.

---

## Prinsip Verifikasi & Batas Tanggung Jawab

* **Validasi Nyata Sebelum Status Selesai:** Sebuah fase atau tugas hanya boleh ditandai selesai (`[x]`) setelah perintah uji atau keluaran aplikasinya benar-benar dijalankan dan diamati hasilnya, bukan sekadar karena proses kompilasi berakhir tanpa pesan kesalahan.
* **Pemisahan Repositori:**
  * `Luciansvon/CalX-weight-tester` menyimpan kode sumber aplikasi CalX (`calx-core`, `calx-cli`, `calx-ui`, skrip `CalX Link`), konfigurasi spesifik proyek, dokumentasi, dan kasus uji proyek.
  * `Luciansvon/B.I.M.A-DEV-INFRA` menyimpan *reusable workflow*, standar audit, benchmark lintas proyek, dan validasi rilis bersama.
