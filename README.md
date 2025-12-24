# Subsi-Gas

Sistem distribusi LPG subsidi yang memvisualisasikan titik penyalur di peta, memantau stok tabung, serta membantu admin dan distributor mengelola rantai pasok secara transparan. Proyek ini dibangun dengan Laravel 12 + Livewire (Flux UI) sehingga seluruh dashboard bersifat reactive tanpa harus menulis JavaScript manual.

## Daftar Isi

1. [Fitur Utama](#fitur-utama)
2. [Preview & Screenshots](#preview--screenshots)
3. [Stack Teknologi](#stack-teknologi)
4. [Persyaratan](#persyaratan)
5. [Setup Cepat](#setup-cepat)
6. [Menjalankan Aplikasi](#menjalankan-aplikasi)
7. [Akun Seed Default](#akun-seed-default)
8. [Struktur Modul & Routing](#struktur-modul--routing)
9. [Arsitektur & Alur Data](#arsitektur--alur-data)
10. [Peta & Pencarian Lokasi Terdekat](#peta--pencarian-lokasi-terdekat)
11. [Alur Undangan Distributor & Email](#alur-undangan-distributor--email)
12. [Testing & Quality](#testing--quality)
13. [Troubleshooting](#troubleshooting)
14. [Deployment](#deployment)
15. [Kontribusi](#kontribusi)
16. [Lisensi](#lisensi)

## Fitur Utama

1. **Peta publik interaktif** – Landing page menampilkan lokasi penyalur gas subsidi beserta stok, jam operasional, foto, dan kontak.
2. **Dashboard Admin** – Mengelola distributor, mengimpor lokasi massal, mengekspor laporan stok, dan memantau status aktif.
3. **Dashboard Distributor** – Distributor dapat menambah/memperbarui lokasi, melampirkan foto, serta mencatat perubahan stok.
4. **Pencarian lokasi terdekat** – Query Haversine di MySQL (fallback perhitungan PHP untuk SQLite) membantu warga menemukan penyalur dalam radius tertentu.
5. **Alur undangan** – Distributor baru dibuat oleh admin dan menerima tautan reset password melalui email (tanpa mengirim kata sandi plaintext).
6. **Integrasi peta fleksibel** – Leaflet memakai tile Mapbox bila token tersedia atau fallback ke OpenStreetMap tanpa biaya.

## Preview & Screenshots

Simpan file gambar pada direktori `docs/screenshots` (buat bila belum ada) agar rapi di versi kontrol. Gunakan rasio 16:9 atau 4:3 dengan resolusi minimal 1280px agar detail UI tetap jelas.

| Halaman              | Deskripsi singkat                         | Preview                                                                 |
|----------------------|-------------------------------------------|-------------------------------------------------------------------------|
| Landing Map Publik   | Lokasi penyalur di peta + filter stok     |<img width="1920" height="1080" alt="Screenshot 2025-12-25 002130" src="https://github.com/user-attachments/assets/4938ed88-6f79-45d1-92a7-f52879c1ae35" />
| Dashboard Admin      | Ringkasan stok, tabel distributor, ekspor | ![Admin Placeholder](docs/screenshots/admin-dashboard.png)             |
| Dashboard Distributor| CRUD lokasi & update stok                | ![Distributor Placeholder](docs/screenshots/distributor-dashboard.png) |

> Ganti tautan gambar di atas dengan screenshot aktual setelah diunggah.

## Stack Teknologi

- Laravel 12, Fortify, Policies & Middleware role-based
- Livewire + Flux UI components + TailwindCSS + Vite
- MySQL (produksi) / SQLite (pengujian) + Eloquent
- Leaflet.js untuk visualisasi peta
- Queue worker (opsional) untuk email undangan

## Persyaratan

- PHP 8.2+
- Composer
- Node.js + npm
- MySQL / MariaDB

## Setup Cepat

```bash
composer install
npm install
```

1. **Salin / buat `.env`**  
   Repo menyediakan `.env.example`. Minimal variabel:

   ```env
   APP_NAME="Subsi-Gas"
   APP_ENV=local
   APP_KEY=
   APP_DEBUG=true
   APP_URL=http://localhost

   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=subsi_gas
   DB_USERNAME=root
   DB_PASSWORD=

   # Opsional (Leaflet tetap jalan dengan OSM tanpa token)
   MAPBOX_TOKEN=
   GOOGLE_MAPS_API_KEY=
   ```

2. **Generate app key**

   ```bash
   php artisan key:generate
   ```

3. **Migrasi + seed**

   ```bash
   php artisan migrate --seed
   ```

4. **Link storage (unggah foto lokasi)**

   ```bash
   php artisan storage:link
   ```

## Menjalankan Aplikasi

- Jalur terpisah:

  ```bash
  npm run dev
  php artisan serve
  ```

- Atau gunakan skrip `composer run dev` untuk menjalankan server, queue listener, dan Vite secara paralel.

## Akun Seed Default

- Seluruh akun seeded memakai kata sandi `password`.
- **Admin:** `admin@example.com` (bisa diubah lewat env `ADMIN_EMAIL`).
- **Distributor:** dibuat otomatis (3 akun contoh). Admin dapat menambah distributor baru melalui dashboard.

## Struktur Modul & Routing

| Peran        | Route utama                     | Komponen Livewire                                  |
|--------------|----------------------------------|----------------------------------------------------|
| Publik       | `/`                              | `App\Livewire\Public\LandingMap`                   |
| Auth umum    | `/dashboard`                     | Redirect peran via middleware `auth`, `active`     |
| Admin        | `/admin/...`                     | Dashboard, manajemen distributor & lokasi, ekspor  |
| Distributor  | `/distributor/...`               | Tabel lokasi, CRUD lokasi, ringkasan stok          |
| Pengaturan   | `/settings/profile|password|...` | Profil, kata sandi, preferensi tampilan            |

Middleware `role:admin` dan `role:distributor` memastikan akses sesuai peran.

## Arsitektur & Alur Data

- **Lapisan presentasi**: Blade minimal + komponen Livewire untuk interaksi reaktif tanpa JavaScript kustom.
- **Lapisan domain**: Model `Location`, `Distributor`, `StockMutation` memakai Policy untuk membatasi aksi berdasarkan peran.
- **Integrasi pihak ketiga**:
  - Map tiles via Mapbox/OSM.
  - Email melalui SMTP (bisa diganti Ses, Mailgun, dll).
- **Pencatatan stok**: Mutasi stok dicatat agar audit trail tersedia dan bisa diekspor.
- **Queue**: Event undangan dan email berat dapat dipindahkan ke queue (`database`/`redis`) agar UI tetap responsif.

## Peta & Pencarian Lokasi Terdekat

```sql
SELECT id, name, address, latitude, longitude, stock,
  (6371 * acos(
    cos(radians(:lat)) * cos(radians(latitude))
    * cos(radians(longitude) - radians(:lng))
    + sin(radians(:lat)) * sin(radians(latitude))
  )) AS distance
FROM locations
HAVING distance <= :radius
ORDER BY distance ASC;
```

Saat testing dengan SQLite in-memory, jarak dihitung ulang di PHP (`Location::haversineDistanceKm`).  
Jika `MAPBOX_TOKEN` kosong, Leaflet otomatis memakai OpenStreetMap tiles.

## Alur Undangan Distributor & Email

1. Admin membuat distributor dari dashboard.
2. Aplikasi mengirim tautan reset password; distributor menetapkan kredensial sendiri.
3. Pastikan konfigurasi mail di `.env` terisi:

   ```env
   MAIL_MAILER=smtp
   MAIL_HOST=smtp.example.com
   MAIL_PORT=587
   MAIL_USERNAME=
   MAIL_PASSWORD=
   MAIL_ENCRYPTION=tls
   MAIL_FROM_ADDRESS=no-reply@example.com
   MAIL_FROM_NAME="${APP_NAME}"
   ```

## Testing & Quality

```bash
php artisan test
```

Gunakan `./vendor/bin/pest` jika ingin menjalankan Pest langsung. Jalankan `php artisan config:clear` sebelum test bila mengganti env.

## Troubleshooting

1. **Gagal load foto lokasi** – pastikan `storage/app/public` sudah di-link dan direktori `storage/logs` writable.
2. **Email tidak terkirim** – cek kredensial SMTP dan jalankan queue worker (`php artisan queue:listen`) bila memakai driver `database`/`redis`.
3. **Map kosong** – pastikan tabel `locations` berisi seed dan browser mengizinkan geolocation (untuk pencarian radius).

## Deployment

1. **Build aset produksi**
   ```bash
   npm run build
   ```
2. **Optimasi Laravel**
   ```bash
   php artisan config:cache
   php artisan route:cache
   php artisan event:cache
   ```
3. **Konfigurasi server**
   - PHP-FPM 8.2+, `memory_limit` minimal 256M.
   - Point document root ke `public/`.
   - Pastikan direktori `storage` dan `bootstrap/cache` writable.
4. **Scheduler & Queue**
   - Tambahkan cron `* * * * * php /path/artisan schedule:run`.
   - Jalankan queue worker (Supervisor / systemd) bila memakai driver async.

## Kontribusi

1. Fork repo & buat branch feature/bugfix baru.
2. Pastikan lint & test lulus sebelum membuat PR.
3. Sertakan deskripsi perubahan dan tangkapan layar bila mengubah UI.
4. Untuk isu besar, buka diskusi/issue terlebih dahulu agar rancangan selaras roadmap.

## Lisensi

MIT License mengikuti lisensi default Laravel starter kit.
