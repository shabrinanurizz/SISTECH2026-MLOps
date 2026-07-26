# Crime Risk Score Prediction & Continual Learning Pipeline

Dataset: `features_labels.csv` dari Hands-On 1, 457.696 baris × 14 kolom (agregat per sel grid × hari × jam). Split train/test: 366.156 / 91.540 baris (±20% holdout, dibekukan sebagai `holdout_df` dan tidak pernah ikut training di seluruh proses continual learning).

## 1. Ringkasan Hasil Evaluasi

| Model | MAE | RMSE | R² |
|---|---|---|---|
| Baseline: Global Mean | 3.742 | 4.958 | -0.000 |
| Baseline: Rata-rata per Sel (`cell_id`) | 3.275 | 4.471 | 0.187 |
| Baseline: Rata-rata per Jam+Hari | 3.468 | 4.635 | 0.126 |
| Baseline: Rata-rata per (Sel, Jam) | 3.630 | 4.916 | 0.017 |
| Baseline: Rata-rata per (Sel, Hari) | 3.657 | 4.960 | -0.001 |
| Model: Linear Regression | 3.042 | 4.181 | 0.289 |
| **Model: Random Forest** | **1.278** | **2.307** | **0.783** |
| Model: Gradient Boosting | 2.790 | 3.835 | 0.402 |

**Analisis :**

- Baseline terkuat ternyata bukan kombinasi `(Sel, Jam)` atau `(Sel, Hari)` seperti dugaan awal, melainkan **Rata-rata per Sel saja** (MAE 3.275, R² 0.187). Ini indikasi kuat bahwa Risk Score didominasi variasi **antar-lokasi**, bukan variasi jam/hari dalam sel yang sama — begitu kombinasi ditambah jam/hari, data per grup jadi terlalu sparse (banyak kombinasi hanya punya sedikit sampel di train), sehingga baseline itu justru *lebih buruk* daripada baseline per-sel saja (MAE 3.630 dan 3.657, mendekati performa Global Mean).   

- Model Linear Regression (MAE 3.042) hanya sedikit lebih baik dari baseline per-sel — wajar, karena `FEATURE_COLS` dasar (`lat_r`, `lon_r`, waktu siklikal, `crime_count`, dst.) belum menangkap identitas sel spesifik seakurat baseline "rata-rata historis per sel".    

- **Random Forest jauh mengungguli semua kandidat lain** (MAE 1.278, R² 0.783) — hampir 2× lebih baik dari baseline terbaik dan dari Linear Regression. Ini terjadi setelah menambahkan fitur `cell_target_enc` (target encoding `cell_id`, dihitung dari `train_df` saja untuk menghindari leakage), yang menurut analisis korelasi (`corr_with_target`) adalah fitur paling berkorelasi dengan `risk_score` (r = 0.619) — jauh di atas fitur lain (`lon_r` r=0.225, `hour_sin` r=-0.194, dst.). `feature_importance_df` dari Random Forest mengonfirmasi hal yang sama: `cell_target_enc` menyumbang **45,2%** dari total importance, diikuti `lat_r` (12,8%) dan `lon_r` (10,9%).    

- Gradient Boosting (MAE 2.790) kalah jauh dari Random Forest dengan hyperparameter default (`n_estimators=300, learning_rate=0.05, max_depth=3`) — kemungkinan *underfit* karena `learning_rate` kecil dan `max_depth` dangkal belum cukup untuk menangkap sinyal kuat dari `cell_target_enc` dalam 300 iterasi; berpotensi membaik dengan tuning lebih lanjut (menaikkan `max_depth` atau `n_estimators`).    


## 2. Narasi Continual Learning: Perjalanan Model

Simulasi menggunakan `N_BATCHES = 5` batch berbasis pengacakan **terstratifikasi per `cell_id`** (bukan pengacakan polos), dengan drift buatan disuntikkan mulai `DRIFT_START_BATCH = 3` (`crime_count × 1.6` + noise, `risk_score × 1.3`). Ukuran batch: Batch 0 = 90.513 baris (37.160 sel unik), Batch 1 = 80.224 (31.699 sel), Batch 2 = 71.909 (27.704 sel), Batch 3 = 64.799 (24.451 sel, **drift buatan**), Batch 4 = 58.711 (21.695 sel, **drift buatan**).   

Model utama: **Random Forest** (`n_estimators=300, min_samples_leaf=3`), strategi training **`sliding_window`** (hanya 3 batch terakhir), kriteria promosi ganda: MAE membaik ≥ margin **dan** R² tidak turun > 0,01.

**Ringkasan siklus versi model (versi final, dengan sliding window + kriteria ganda):**

| Versi | Batch | Ukuran Train | Drift? | MAE Kandidat | R² Kandidat | vs Champion (MAE / R²) | Keputusan |
|---|---|---|---|---|---|---|---|
| v0 | 0 | 90.513 | – | 2,451 | 0,452 | – (champion awal) | `initial_champion` |
| v1 | 1 | 170.737 | Ya (`risk_score` PSI 0,001, concept-drift residual) | **2,177** | **0,521** | 2,451 / 0,452 | **`promoted`** |
| v2 | 2 | 242.646 | Ya (`crime_count` & `risk_score` PSI naik tipis, concept-drift residual) | **1,984** | **0,569** | 2,177 / 0,521 | **`promoted`** |
| v3 | 3 | 216.932 | Ya, drastis (`crime_count` PSI 13,68, `risk_score` PSI 4,79) | 2,990 | 0,079 | 1,984 / 0,569 | `kept_champion` |
| v4 | 4 | 195.419 | Ya, drastis (`crime_count` PSI 8,65, `risk_score` PSI 2,56) | 3,668 | -0,299 | 1,984 / 0,569 | `kept_champion` |

**Champion akhir: MAE holdout 1,984, R² 0,569** (versi v2) — bertahan sampai akhir simulasi.

**Narasi:**

Model awal (v0) dilatih dari Batch 0 sebelum drift apa pun terjadi, menghasilkan MAE holdout 2,451. Di Batch 1 dan Batch 2, sistem drift detection mendeteksi pergeseran ringan namun nyata (nilai PSI di kisaran 0,001–0,002, bukan drift ekstrem — didorong terutama oleh **concept drift pada residual model**, bukan perubahan distribusi fitur mentah). Kandidat model yang dilatih ulang di kedua batch ini **berhasil dipromosikan**, karena MAE-nya konsisten membaik (2,451 → 2,177 → 1,984) dan R² juga naik (0,452 → 0,521 → 0,569) — sinyal bahwa data baru memang membawa informasi berguna, bukan sekadar noise.

Titik krusial terjadi di Batch 3, tempat drift buatan pertama kali disuntikkan (`crime_count` naik ~60%, `risk_score` naik ~30%). PSI melonjak drastis (13,68 untuk `crime_count`, jauh di atas ambang 0,25), dan kandidat model yang dilatih dari jendela 3-batch-terakhir (kini didominasi data yang sudah "terdistorsi" drift buatan) menghasilkan MAE 2,990 dan **R² anjlok ke 0,079** — jelas lebih buruk dari champion (MAE 1,984, R² 0,569). Sistem **menahan promosi** (`kept_champion`) sesuai kriteria ganda: MAE tidak membaik dan R² jatuh jauh di bawah margin toleransi. Ini keputusan yang tepat — kandidat itu overfit ke pola drift buatan yang sifatnya sintetis/ekstrem, bukan pola nyata yang layak dipelajari model produksi.

Pola yang sama berulang di Batch 4: PSI masih tinggi (8,65), kandidat makin buruk (MAE 3,668, **R² negatif** -0,299, artinya model ini lebih buruk daripada sekadar menebak rata-rata), dan kembali ditolak.

**Kesimpulan model journey:** champion yang bertahan sampai akhir adalah **v2** (dilatih dari Batch 0–2, sebelum drift buatan disuntikkan). Continual learning di sini berhasil menunjukkan fungsi **paling pentingnya**: bukan hanya mempromosikan model yang membaik, tapi **menahan** model yang memburuk drastis — mencegah model produksi diganti oleh kandidat yang justru rusak akibat perubahan data ekstrem (dalam kasus nyata, drift seburuk ini sebaiknya juga memicu investigasi manual terhadap sumber datanya, bukan cuma retrain otomatis).


## 3. Justifikasi Threshold & Kriteria Retrain

- **Deteksi drift** memakai tiga sinyal digabung dengan logika **OR**: KS test (`alpha = 0,01`, cukup ketat untuk menghindari retrain karena fluktuasi sampling), PSI (`threshold = 0,25`, standar umum industri: <0,1 stabil, 0,1–0,25 sedang, >0,25 signifikan), dan **concept drift** berbasis pergeseran distribusi residual champion model. Concept drift ini terbukti berguna: di Batch 1 dan 2, PSI pada fitur mentah sangat kecil (0,0005–0,002, jauh di bawah 0,25) sehingga **tidak akan terdeteksi** kalau hanya mengandalkan PSI/KS pada fitur — tapi residual model tetap bergeser signifikan (KS-stat 0,132 dan 0,116, p≈0), menandakan hubungan fitur→label sudah mulai berubah walau distribusi datanya sendiri masih terlihat stabil. Ini membuktikan pentingnya sinyal ketiga ini, bukan sekadar tambahan.   

- **Kriteria promosi ganda** (MAE membaik **dan** R² tidak turun >0,01) terbukti krusial di Batch 3–4: kalau kriterianya hanya "MAE kandidat ≤ MAE champion", kandidat di Batch 3 (MAE 2,990) memang masih lebih buruk dari champion (1,984) jadi tetap tertolak — tapi kriteria R² memberi lapisan keamanan tambahan yang eksplisit menangkap kasus model "gagal total" (R² negatif di Batch 4), bukan cuma "kurang optimal".    

- **Strategi `sliding_window` (3 batch terakhir)** dipilih dibanding melatih dari seluruh data kumulatif, dengan asumsi data lama yang mencerminkan pola sebelum perubahan bisa mengencerkan sinyal terbaru. Namun hasil aktual menunjukkan trade-off ini menggigit balik: begitu drift buatan masuk ke jendela (Batch 3–4), *seluruh* data training kandidat langsung didominasi data yang terdistorsi (bukan tercampur dengan data lama yang lebih stabil), sehingga performa kandidat jatuh drastis. Jendela yang lebih panjang (mis. 4–5 batch) kemungkinan akan membuat kandidat lebih tahan terhadap drift ekstrem sesaat karena tercampur data lama — ini area yang belum diuji dan layak jadi eksperimen lanjutan.

## 4. Refleksi: Kendala & Solusi

- **Kendala — tidak ada kolom `Datetime` per-kejadian** di dataset agregat. Simulasi batch tidak bisa merepresentasikan urutan waktu asli.   
  **Solusi:** batch dibuat dengan pengacakan **terstratifikasi per `cell_id`** (bukan acak polos) agar tiap batch tetap merepresentasikan sebaran ~20-37 ribu sel yang konsisten proporsinya, dan keterbatasan ini didokumentasikan eksplisit — bukan diklaim sebagai simulasi waktu nyata.

- **Kendala — kandidat model runtuh total saat drift ekstrem (R² negatif di Batch 4).** Ini sempat terlihat seperti bug, tapi setelah ditelusuri lewat laporan drift (`PSI` 8,65 untuk `crime_count`), ternyata memang konsekuensi logis dari drift buatan yang sangat agresif (+60% crime_count, +30% risk_score) mendominasi seluruh jendela training 3-batch.   
  **Solusi:** bukan diperbaiki dengan mengubah model, melainkan **dibiarkan sebagai temuan valid** dan justru jadi bukti bahwa mekanisme `kept_champion` bekerja sebagaimana mestinya — sesuai arahan notebook bahwa continual learning dinilai dari kejujuran narasi, bukan kesempurnaan angka.