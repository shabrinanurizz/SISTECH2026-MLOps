# Chicago Crime Risk Scoring

## Dataset

Chicago Crimes Dataset berisi catatan kejadian kriminal di Kota Chicago yang mencakup informasi waktu kejadian, jenis kejahatan, lokasi, status penangkapan, serta koordinat geografis. Dataset digunakan untuk membangun Risk Score berbasis informasi temporal dan spasial.

Subset yang digunakan pada eksperimen ini mengambil **3 tahun data terakhir** yang tersedia (`Year >= last_year - 2`), dengan justifikasi bahwa pola kejahatan pada rentang tahun terbaru lebih relevan untuk memprediksi risiko saat ini dibanding data yang terlalu lama.

---

## Design Decisions

- **Severity Scoring:** Setiap kombinasi `Primary Type` + `Description` diberi bobot keparahan (0--100) melalui skema dua level: (1) skor dasar per `Primary Type` yang mencakup hampir seluruh kategori kejahatan pada dataset (bukan satu nilai default global), dan (2) modifier berbasis kata kunci pada `Description` (mis. `ARMED`/`AGGRAVATED` menaikkan skor, `ATTEMPT` menurunkannya, nominal kerugian finansial menyesuaikan skor kejahatan properti). Pendekatan ini secara signifikan mengurangi persentase kejadian yang jatuh ke nilai fallback dibanding skema satu-default.
- **Temporal Relevance:** Informasi waktu dimodelkan melalui dua mekanisme:
  - *Cyclical encoding* (`sin`/`cos`) pada `hour` dan `dow` agar sifat siklikal waktu (mis. jam 23:00 dekat dengan 00:00) terepresentasi dengan benar, bukan sebagai jarak numerik linear.
  - *Temporal decay* eksponensial dengan *half-life* 180 hari (~6 bulan) --- relevansi sebuah kejadian berkurang separuh setiap 6 bulan, namun tidak pernah benar-benar nol seperti pada peluruhan linear. Parameter ini dipilih dengan asumsi pola kejahatan di suatu lokasi cukup persisten dalam skala bulanan--tahunan.
- **Spatial Relevance:** Lokasi direpresentasikan melalui grid aggregation dengan resolusi 3 desimal koordinat (~110m per sel) --- lebih halus dibanding baseline (2 desimal, ~1.1km) agar risiko lebih spesifik terhadap lokasi. Untuk mengatasi sparsity akibat grid yang lebih halus, dilakukan *spatial smoothing* menggunakan jarak geografis sesungguhnya (`BallTree` dengan metrik haversine) dan bobot kernel Gaussian (radius 3km, bandwidth 1.5km) per irisan hari×jam yang sama --- menggantikan pendekatan tetangga grid 3×3 tanpa bobot jarak.
- **Feature Representation:** Fitur akhir terdiri dari fitur temporal (`hour`, `dow`, serta bentuk cyclical-nya), fitur spasial (`cell_id`, `lat_r`, `lon_r`), dan fitur agregat (`crime_count`, serta opsional `arrest_rate` dan `violent_share`). Fitur-fitur ini dikombinasikan dengan severity dan temporal decay melalui pseudo-labeling untuk membentuk Risk Score, yang kemudian dinormalisasi menggunakan `log1p + min-max` (dipilih setelah dibandingkan dengan percentile rank dan clipping-based scaling, karena distribusinya paling simetris dan tetap mempertahankan urutan magnitude asli).

---

## EDA Insights

> Catatan: poin di bawah adalah kerangka insight berdasarkan pola umum yang biasanya muncul pada data ini. Isi angka/persentase spesifik (ditandai `[...]`) perlu dilengkapi dari hasil run notebook masing-masing, karena bergantung pada subset tahun dan area yang dipakai.

- **Jenis kejahatan paling dominan:** Berdasarkan `value_counts()` pada `Primary Type`, kategori seperti THEFT, BATTERY, dan CRIMINAL DAMAGE umumnya mendominasi jumlah kejadian, sementara kategori berat seperti HOMICIDE atau CRIM SEXUAL ASSAULT jauh lebih jarang tapi punya dampak keparahan tertinggi. *(Isi: 3 jenis kejahatan teratas beserta jumlah/persentasenya = `[...]`)*
- **Pola temporal:**
  - Kejadian cenderung meningkat pada jam malam hingga dini hari untuk kejahatan berbasis kekerasan (BATTERY, ASSAULT), sedangkan kejahatan properti (THEFT, DECEPTIVE PRACTICE) lebih merata sepanjang siang hari.
  - Pola akhir pekan vs hari kerja menunjukkan perbedaan --- akhir pekan cenderung memiliki puncak kejadian yang bergeser lebih larut malam dibanding hari kerja. *(Isi: jam puncak spesifik hasil observasi = `[...]`)*
  - Heatmap interaksi hari×jam menunjukkan kombinasi waktu tertentu (mis. malam Jumat--Sabtu) memiliki konsentrasi kejadian yang jauh lebih tinggi dibanding kombinasi lain --- pola yang tidak terlihat bila hari dan jam dianalisis terpisah.
- **Pola spasial:** Peta hexbin menunjukkan kejahatan tidak tersebar merata --- terdapat beberapa *hotspot* dengan kepadatan jauh lebih tinggi dari area sekitarnya, umumnya terkonsentrasi di area dengan aktivitas komersial/publik tinggi. *(Isi: nama area/koordinat hotspot dominan = `[...]`)*
- **Interaksi jenis kejahatan × waktu:** Beberapa jenis kejahatan punya "jam sibuk" yang berbeda satu sama lain (dilihat dari distribusi proporsi per jam untuk 5 jenis kejahatan teratas) --- insight ini memvalidasi kebutuhan fitur temporal *dan* interaksinya dalam pemodelan Risk Score, bukan hanya frekuensi total.

---

## Challenges & Reflection

**Challenges**
- Dataset berukuran besar sehingga membutuhkan optimasi penggunaan memori.
- Adanya missing value dan koordinat tidak valid (mis. 0,0) pada beberapa atribut lokasi.
- Menentukan skema severity scoring yang representatif --- baseline dengan satu nilai default global menyebabkan sebagian besar data kehilangan informasi keparahan yang bermakna.
- Grid spasial yang terlalu kasar (2 desimal) kurang presisi terhadap lokasi, sementara grid yang lebih halus (3 desimal) menimbulkan banyak sel sepi kejadian (sparse), sehingga smoothing antar-sel menjadi krusial.
- Normalisasi min--max sederhana rentan terhadap outlier, di mana beberapa sel dengan risiko ekstrem menekan skor seluruh sel lain ke rentang yang sempit.

**Solutions**
- Membaca hanya kolom yang diperlukan (`usecols`) serta membatasi subset ke 3 tahun terakhir untuk menghemat memori dan komputasi.
- Melakukan pembersihan data (drop missing value, filter bounding box Chicago) sebelum feature engineering.
- Menetapkan severity score dua level (skor dasar per `Primary Type` + modifier kata kunci `Description`) untuk memperluas cakupan dan mengurangi ketergantungan pada nilai default.
- Memperhalus grid spasial menjadi 3 desimal, lalu mengatasi sparsity-nya dengan spatial smoothing berbasis jarak geografis sesungguhnya (BallTree haversine + kernel Gaussian) per irisan hari×jam.
- Menggunakan temporal decay eksponensial (half-life) alih-alih linear agar kejadian lama tetap memiliki relevansi kecil, bukan langsung nol.
- Membandingkan beberapa metode normalisasi (min-max, log1p, percentile rank, clipping) secara visual sebelum memilih `log1p + min-max` sebagai metode final, karena menghasilkan distribusi Risk Score paling simetris tanpa menghilangkan informasi magnitude.