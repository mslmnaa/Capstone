# Panduan Superset — Dataset & Dashboard (Capstone GMEDIA)

Panduan langkah demi langkah membangun dashboard **Customer 360 GMEDIA** di Apache Superset / Preset,
setelah tabel berhasil di-push ke Postgres online.

- **Database**: sudah terkoneksi (Postgres/Neon).
- **Prinsip**: dua sumber data **tidak di-join** — masuk sebagai dataset terpisah, digabung di **level dashboard**.
  - `fact_interaksi` → sisi **Operasional** (log interaksi CS).
  - `fact_review_publik` → sisi **Suara Pelanggan** (review Maps, sumber sentimen).

---

## 0 · Tabel yang tersedia di database

| Tabel | Grain | Kolom penting |
|---|---|---|
| `fact_interaksi` | 1 interaksi | `report_start` (waktu), `intent`, `problem`, `cs_response`, `outcome_status`, `is_resolved_explicit`, `dur_min`, `source`, `cid_segmentation`, `sid_product_name`, `account_manager_name` |
| `fact_review_publik` | 1 review | `brand`, `stars`, `sentimen`, `confidence`, `text` |
| `pola_resolusi` | 1 problem | `volume`, `response_dominan`, `median_durasi_mnt`, `p90_durasi_mnt`, `pct_selesai_eksplisit` |
| `sla_problem` | 1 problem | `median_mnt`, `p90_mnt`, `volume` |
| `perangkat_kronis` | 1 perangkat | `cid_name`, `sid_product_name`, `n_down`, `median_durasi_mnt` |
| `tren_bulanan` | 1 bulan | `month`, `volume`, `pct_gangguan` |
| `profil_segmen` | 1 segmen | `persona`, `jml_pelanggan`, `rata_interaksi` |
| `dim_segmen` | 1 pelanggan | `cid_name`, `persona`, `n_interaksi` |
| `keyword_problem` | 1 problem | `keywords` |

---

## 1 · Membuat Dataset

Superset: menu atas **Datasets → + Dataset** (kanan atas).

1. **Database**: pilih koneksi Postgres Anda.
2. **Schema**: `public`.
3. **Table**: pilih tabelnya → **Create Dataset and Create Chart** (atau *Add*).
4. Ulangi untuk setiap tabel yang dibutuhkan. Minimal wajib: **`fact_interaksi`** dan **`fact_review_publik`**. Tambahkan `sla_problem`, `perangkat_kronis`, `tren_bulanan`, `profil_segmen` untuk chart pendukung.

### 1a · Set kolom waktu (penting untuk time-series)
Untuk `fact_interaksi`: buka dataset → tab **Columns** → cari `report_start` → aktifkan **Is temporal** (biasanya sudah otomatis karena tipe `timestamp`). Ini yang membuat filter waktu & chart tren bekerja.
Untuk `tren_bulanan`: pastikan `month` juga `Is temporal`.

### 1b · Metric kustom (buat sekali, dipakai ulang di banyak chart)
Buka dataset → tab **Metrics → + Add item**. Isi *Metric name*, *SQL expression*, *Label*.

**Dataset `fact_interaksi`:**
| Metric name | SQL expression | Arti |
|---|---|---|
| `jml_interaksi` | `COUNT(*)` | total interaksi |
| `pct_selesai` | `AVG(is_resolved_explicit) * 100` | % selesai eksplisit |
| `median_durasi` | `percentile_cont(0.5) WITHIN GROUP (ORDER BY dur_min)` | median menit (khas Postgres) |
| `p90_durasi` | `percentile_cont(0.9) WITHIN GROUP (ORDER BY dur_min)` | p90 menit |

**Dataset `fact_review_publik`:**
| Metric name | SQL expression | Arti |
|---|---|---|
| `jml_review` | `COUNT(*)` | total review |
| `avg_bintang` | `AVG(stars)` | rata-rata bintang |
| `pct_negatif` | `AVG(CASE WHEN sentimen='Negatif' THEN 1 ELSE 0 END) * 100` | % review negatif |

> Catatan: `percentile_cont` hanya jalan di Postgres (bukan SQLite). Karena Anda sudah pindah ke Postgres online, ini aman.

---

## 2 · Membuat Chart

Menu atas **Charts → + Chart** → pilih **Dataset** → pilih **Chart type** → atur di panel **Data** → **Create chart** → **Save** (beri nama).

### BAGIAN 1 — Operasional (dataset `fact_interaksi`, kecuali disebut lain)

| # | Nama chart | Chart type | Dataset | Metric | Dimension / Kolom | Catatan |
|---|---|---|---|---|---|---|
| 1 | Total Interaksi | **Big Number** | fact_interaksi | `jml_interaksi` | — | KPI |
| 2 | % Selesai Eksplisit | **Big Number** | fact_interaksi | `pct_selesai` | — | KPI |
| 3 | Median Durasi (mnt) | **Big Number** | fact_interaksi | `median_durasi` | — | KPI |
| 4 | Volume per Bulan | **Time-series Line** | tren_bulanan | `SUM(volume)` | Time = `month` | tren utama |
| 5 | Distribusi Intent | **Bar Chart** | fact_interaksi | `jml_interaksi` | Dimensions = `intent` | urut desc |
| 6 | Problem Category | **Bar Chart** | fact_interaksi | `jml_interaksi` | Dimensions = `problem` | |
| 7 | Sumber Interaksi | **Pie Chart** | fact_interaksi | `jml_interaksi` | Dimensions = `source` | auto vs pelanggan |
| 8 | Response CS | **Bar Chart** | fact_interaksi | `jml_interaksi` | Dimensions = `cs_response` | |
| 9 | Status Outcome | **Bar Chart** | fact_interaksi | `jml_interaksi` | Dimensions = `outcome_status` | 3-state |
| 10 | SLA per Problem | **Bar Chart** | sla_problem | `MAX(median_mnt)` & `MAX(p90_mnt)` | Dimensions = `problem` | 2 metric berdampingan |
| 11 | Problem × Intent | **Heatmap** | fact_interaksi | `jml_interaksi` | X = `problem`, Y = `intent` | |
| 12 | Perangkat Kronis (Top 15) | **Table** | perangkat_kronis | `MAX(n_down)` | Group by `cid_name`,`sid_product_name` | Sort `n_down` desc, Row limit 15 |
| 13 | Persona Pelanggan | **Bar Chart** | profil_segmen | `SUM(jml_pelanggan)` | Dimensions = `persona` | |

### BAGIAN 2 — Suara Pelanggan (dataset `fact_review_publik`)

| # | Nama chart | Chart type | Metric | Dimension / Kolom | Catatan |
|---|---|---|---|---|---|
| 14 | Total Review | **Big Number** | `jml_review` | — | KPI |
| 15 | Rata-rata Bintang | **Big Number** | `avg_bintang` | — | KPI |
| 16 | % Review Negatif | **Big Number** | `pct_negatif` | — | KPI |
| 17 | Distribusi Sentimen | **Pie Chart** | `jml_review` | Dimensions = `sentimen` | Positif/Netral/Negatif |
| 18 | Distribusi Bintang | **Bar Chart** | `jml_review` | Dimensions = `stars` | 1–5 |
| 19 | Sentimen × Brand | **Bar Chart (stacked)** | `jml_review` | Dimensions = `brand`, Breakdown/Series = `sentimen` | Fiberstream vs GMEDIA |
| 20 | Contoh Review Negatif | **Table** | — (raw records) | Columns = `brand`,`stars`,`text` | Filter `sentimen = Negatif`, limit 20 |
| 21 | Kata Sering Muncul | **Word Cloud** | `jml_review` | Series = `text` | opsional |

> Tip mengisi dua metric di chart bar (mis. #10): di panel **Metrics** klik **+ Add metric** dua kali (median & p90).

---

## 3 · Menyusun Dashboard

Menu atas **Dashboards → + Dashboard**.

1. Beri judul: **Customer 360 GMEDIA**.
2. Dari panel kanan (daftar chart), **drag** chart ke kanvas. Susun jadi dua blok:

```
┌──────────────────────────── Customer 360 GMEDIA ─────────────────────────────┐
│ [Filter: Rentang Waktu] [Segmentasi] [Produk] [Account Manager] [Brand]      │
│                                                                              │
│ ── BAGIAN 1 · OPERASIONAL ──                                                 │
│ [Total Interaksi] [% Selesai] [Median Durasi]        ← baris KPI (Big Number)│
│ [ Volume per Bulan  (lebar penuh) ]                                          │
│ [ Distribusi Intent ] [ Problem Category ] [ Sumber ]                        │
│ [ SLA per Problem ] [ Status Outcome ] [ Response CS ]                       │
│ [ Problem × Intent (heatmap) ] [ Perangkat Kronis (tabel) ]                  │
│ [ Persona Pelanggan ]                                                        │
│                                                                              │
│ ── BAGIAN 2 · SUARA PELANGGAN (Review Maps) ──                               │
│ [Total Review] [Avg Bintang] [% Negatif]             ← baris KPI            │
│ [ Distribusi Sentimen ] [ Distribusi Bintang ] [ Sentimen × Brand ]          │
│ [ Contoh Review Negatif (tabel) ] [ Word Cloud ]                            │
└──────────────────────────────────────────────────────────────────────────────┘
```

3. Tambahkan **judul antar-bagian** pakai komponen **Header/Markdown** (drag dari tab *Layout elements*):
   - `## Bagian 1 · Operasional (log interaksi CS)`
   - `## Bagian 2 · Suara Pelanggan (review Google Maps)`
4. Tambahkan satu **Markdown** kecil berisi disclaimer (penting untuk sidang):
   > *Dua kanal ini berasal dari sumber berbeda (log internal vs review publik) dan **tidak ditautkan per-baris** — reviewer dianggap sama secara bisnis, tapi tak ada relasi ID. Penggabungan bersifat kontekstual di dashboard.*

### 3a · Menambahkan Filter dashboard
Klik ikon **Filter** (panel kiri) → **+ Add/Edit Filters**:

| Filter | Tipe | Kolom | Dataset dasar | Berlaku ke |
|---|---|---|---|---|
| Rentang Waktu | **Time range** | `report_start` | fact_interaksi | Bagian 1 |
| Segmentasi | **Value** | `cid_segmentation` | fact_interaksi | Bagian 1 |
| Produk | **Value** | `sid_product_name` | fact_interaksi | Bagian 1 |
| Account Manager | **Value** | `account_manager_name` | fact_interaksi | Bagian 1 |
| Brand | **Value** | `brand` | fact_review_publik | Bagian 2 |

> **Penting:** filter hanya memengaruhi chart yang datasetnya cocok. Jadi *Rentang Waktu/Produk* otomatis hanya mengubah Bagian 1; *Brand* hanya Bagian 2. Ini konsekuensi wajar dari dua dataset terpisah — bukan bug. Di setiap filter, buka **Scoping** dan batasi ke chart bagian yang relevan agar rapi.

5. **Save** dashboard.

---

## 4 · Poin penting & jebakan umum

- **`fact_review_publik` tanpa kolom tanggal** → tak bisa ikut filter waktu; sajikan sebagai potret agregat. Sudah dijelaskan di disclaimer.
- **Median** butuh `percentile_cont` (Postgres) — kalau muncul error, pastikan dataset menunjuk ke koneksi Postgres, bukan SQLite lama.
- **Refresh data**: setiap notebook di-run ulang, tabel di Postgres tertimpa (`if_exists="replace"`). Di Superset klik **Refresh** pada chart/dashboard untuk memuat data terbaru; struktur kolom tak berubah jadi chart tetap valid.
- **Warna sentimen konsisten**: di chart sentimen, set warna manual Positif=hijau, Netral=abu/kuning, Negatif=merah (panel *Customize*).
- **Row limit**: untuk tabel `text` review, batasi 20–50 baris agar ringan.

---

## 5 · Urutan pengerjaan singkat (checklist)

- [ ] Datasets dibuat: `fact_interaksi`, `fact_review_publik`, `sla_problem`, `perangkat_kronis`, `tren_bulanan`, `profil_segmen`
- [ ] `report_start` & `month` = *temporal*
- [ ] Metric kustom dibuat di `fact_interaksi` & `fact_review_publik`
- [ ] 21 chart dibuat & di-save
- [ ] Dashboard "Customer 360 GMEDIA" disusun dua bagian
- [ ] 5 filter ditambah + di-scope ke bagian yang benar
- [ ] Disclaimer markdown ditambahkan
- [ ] Save & test filter
