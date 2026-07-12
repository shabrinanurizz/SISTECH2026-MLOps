# Deskripsi Kolom Dataset Chicago Crime

## Kolom Data Mentah
| Nama Kolom | Deskripsi |
|------------|-----------|
| **ID** | Identitas unik untuk setiap data kejadian. |
| **Case Number** | Nomor laporan (RD Number) yang unik untuk setiap kasus kriminal. |
| **Date** | Tanggal dan waktu terjadinya insiden (dapat berupa estimasi). |
| **Block** | Alamat yang telah disamarkan hingga tingkat blok jalan untuk menjaga privasi. |
| **IUCR** | Kode Illinois Uniform Crime Reporting yang mengidentifikasi jenis kejahatan. |
| **Primary Type** | Kategori utama dari tindak kejahatan. |
| **Description** | Deskripsi lebih rinci atau subkategori dari jenis kejahatan. |
| **Location Description** | Deskripsi lokasi tempat kejadian, misalnya jalan, rumah, sekolah, atau toko. |
| **Arrest** | Menunjukkan apakah telah dilakukan penangkapan terhadap pelaku (Ya/Tidak). |
| **Domestic** | Menunjukkan apakah kasus berkaitan dengan kekerasan dalam rumah tangga (Ya/Tidak). |
| **Beat** | Wilayah patroli polisi tempat kejadian terjadi. |
| **District** | Distrik kepolisian tempat kejadian terjadi. |
| **Ward** | Wilayah administrasi Dewan Kota (City Council District) tempat kejadian terjadi. |
| **Community Area** | Area komunitas di Kota Chicago tempat kejadian terjadi. |
| **FBI Code** | Kode klasifikasi kejahatan berdasarkan standar FBI (NIBRS). |
| **X Coordinate** | Koordinat X lokasi kejadian pada sistem proyeksi State Plane Illinois East NAD 1983. |
| **Y Coordinate** | Koordinat Y lokasi kejadian pada sistem proyeksi State Plane Illinois East NAD 1983. |
| **Year** | Tahun terjadinya insiden. |
| **Updated On** | Tanggal dan waktu terakhir data diperbarui. |
| **Latitude** | Koordinat lintang (latitude) lokasi kejadian yang telah disamarkan. |
| **Longitude** | Koordinat bujur (longitude) lokasi kejadian yang telah disamarkan. |
| **Location** | Lokasi dalam format geospasial yang dapat digunakan untuk visualisasi peta dan analisis spasial. |

## Kolom Yang Digunakan
usecols = ["ID", "Date", "Primary Type", "Description",
           "Location Description", "Arrest", "Latitude", "Longitude", "Year"]