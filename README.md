---

# Dokumentasi Manual Storage Link (Laravel 12 - InfinityFree)

Panduan ini berisi langkah-langkah teknis untuk menghubungkan folder penyimpanan (`storage`) ke folder publik (`htdocs`) pada layanan hosting InfinityFree tanpa menggunakan perintah Terminal/SSH.

## 📝 Masalah
Di InfinityFree, fungsi `php artisan storage:link` tidak dapat dijalankan karena akses SSH dilarang dan fungsi PHP `symlink()` dimatikan. Tanpa ini, file yang diunggah ke folder `storage/app/public` tidak akan bisa diakses oleh browser.

## 🚀 Solusi: Path Binding & Disk Redirect
Kita akan "memaksa" Laravel untuk mengenali folder `htdocs` sebagai folder publik dan mengarahkan penyimpanan langsung ke sana.

---

### Langkah 1: Konfigurasi AppServiceProvider
Buka file `app/Providers/AppServiceProvider.php`. Kita akan melakukan *binding* pada `path.public`.

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        /**
         * FIX: Manual Storage Link untuk InfinityFree
         * Mengarahkan public_path() agar menunjuk ke folder 'htdocs'
         */
        $this->app->bind('path.public', function () {
            return base_path('htdocs');
        });
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        //
    }
}
```

---

### Langkah 2: Konfigurasi Filesystem Disk
Buka file `config/filesystems.php`. Cari bagian `disks` dan sesuaikan array `public` agar sinkron dengan perubahan di langkah pertama.

```php
'disks' => [

    'local' => [
        'driver' => 'local',
        'root' => storage_path('app'),
        'throw' => false,
    ],

    'public' => [
        'driver' => 'local',
        // Mengarah ke htdocs/storage secara otomatis
        'root' => public_path('storage'), 
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
        'throw' => false,
    ],
    
    // Disk lainnya...
],
```

---

### Langkah 3: Pengaturan Manual di File Manager
Karena kita tidak menggunakan *symlink*, kamu harus membuat folder fisik secara manual:
1. Login ke **File Manager** InfinityFree atau gunakan **FTP (FileZilla)**.
2. Masuk ke folder **`htdocs`**.
3. Buat folder baru bernama **`storage`**.
4. (Opsional) Di dalam `htdocs/storage`, buat folder tambahan sesuai kebutuhan aplikasi (misal: `covers`, `avatars`).

---

### Langkah 4: Contoh Implementasi Kode (Controller)
Berikut adalah contoh cara mengunggah file (misalnya sampul buku) menggunakan sistem ini:

```php
public function store(Request $request)
{
    $request->validate([
        'sampul' => 'required|image|mimes:jpeg,png,jpg|max:2048',
    ]);

    if ($request->hasFile('sampul')) {
        // Simpan ke htdocs/storage/book_covers
        $path = $request->file('sampul')->store('book_covers', 'public');
        
        // Simpan nama path ke database
        Book::create([
            'title' => $request->title,
            'cover_path' => $path,
        ]);
    }

    return back()->with('success', 'Buku berhasil ditambahkan!');
}
```

### Langkah 5: Cara Menampilkan File di Blade
Untuk menampilkan file yang sudah diunggah, gunakan *helper* `asset()`:

```html
<img src="{{ asset('storage/' . $book->cover_path) }}" alt="Sampul Buku">
```

---

## 📂 Struktur Folder Akhir
Setelah diterapkan, struktur folder di hosting kamu akan terlihat seperti ini:

```text
/
├── app/
├── config/
├── htdocs/           <-- Folder Publik Utama
│   ├── index.php
│   ├── .htaccess
│   └── storage/      <-- Folder yang kita buat manual
│       └── book_covers/
├── storage/          <-- Folder Storage Asli (Internal)
└── vendor/
```



## ⚠️ Catatan Penting
* **Permission:** Pastikan folder `htdocs/storage` memiliki izin akses **755**.
* **Cache:** Jika gambar tidak muncul setelah perubahan, hapus cache secara manual dengan menghapus semua file di dalam folder `storage/framework/views/` (jangan hapus folder itu sendiri).
* **Environment:** Pastikan variabel `APP_URL` di file `.env` sudah menggunakan domain yang benar (contoh: `https://proyek-kamu.infinityfreeapp.com`).

---
*Dokumentasi ini dibuat untuk keperluan pengembangan aplikasi web di lingkungan hosting terbatas.*
