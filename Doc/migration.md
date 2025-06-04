# Skema Migrasi Database UEU Bootcamp Terintegrasi

Dokumen ini menyajikan seluruh skema migrasi *database* yang diperlukan untuk sistem terintegrasi UEU Bootcamp, meliputi modul LMS dan HRM. Skema ini dirancang berdasarkan kebutuhan fungsional dan non-fungsional yang tertera dalam *Software Requirements Specification* (SRS) dan *Business Requirements Document* (BRD).

Setiap bagian akan mencakup:
1.  **Nama File Migrasi:** Mengikuti konvensi Laravel (`YYYY_MM_DD_HHMMSS_nama_migrasi.php`).
2.  **Kode Migrasi PHP:** Struktur dasar migrasi Laravel untuk membuat atau memodifikasi tabel.
3.  **Penjelasan Tabel dan Kolom:** Detail fungsi setiap tabel dan kolom beserta tipe datanya.
4.  **Catatan BRD & Implementasi:** Keterangan bagaimana skema ini memenuhi kebutuhan BRD, serta potensi klarifikasi atau pertimbangan implementasi di tingkat aplikasi.

**Penting:** Urutan migrasi sangat krusial! Pastikan migrasi yang membuat tabel yang menjadi *foreign key* (kunci asing) dieksekusi terlebih dahulu.

---

## 1. Sistem Inti & Manajemen Pengguna (Core System & User Management)

Modul-modul ini adalah dasar dari seluruh aplikasi, menangani otentikasi, peran pengguna, dan konfigurasi umum.

### `2025_06_05_000001_create_users_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis (BigInt, Unsigned, Auto Increment)
            $table->string('name'); // Nama lengkap pengguna (VARCHAR 255)
            $table->string('email')->unique(); // Alamat email, harus unik (VARCHAR 255)
            $table->timestamp('email_verified_at')->nullable(); // Waktu verifikasi email (TIMESTAMP, bisa NULL)
            $table->string('password'); // Kata sandi pengguna (VARCHAR 255, akan di-hash)
            $table->rememberToken(); // Token untuk fitur "ingat saya" (VARCHAR 100)
            $table->timestamps(); // Kolom `created_at` dan `updated_at` (TIMESTAMP)
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

* **Penjelasan:** Tabel fundamental ini menyimpan semua akun pengguna yang akan mengakses sistem, termasuk admin, pengajar, dan siswa. `email` sebagai `unique` menjamin setiap akun memiliki identitas yang berbeda.
* **Catatan BRD & Implementasi:**
    * **Karakteristik Pengguna:** BRD menyebutkan berbagai jenis pengguna (Admin Company Profile, Admin HRM, Admin LMS, Pengajar, Siswa, System Notification). Tabel `users` ini akan menjadi basis untuk semua jenis pengguna tersebut. Perbedaan peran akan diatur oleh kolom `role_id` (lihat migrasi berikutnya).
    * **Keamanan (Autentikasi):** Kolom `password` akan menyimpan kata sandi yang di-hash, memenuhi kebutuhan keamanan untuk autentikasi.

---

### `2025_06_05_000002_create_roles_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('roles', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('name')->unique(); // Nama peran unik (e.g., 'admin_hrm', 'pengajar', 'siswa') (VARCHAR 255)
            $table->string('display_name')->nullable(); // Nama peran yang lebih mudah dibaca (e.g., 'Administrator HRM') (VARCHAR 255, bisa NULL)
            $table->text('description')->nullable(); // Deskripsi peran (TEXT, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('roles');
    }
};
```

* **Penjelasan:** Tabel ini mendefinisikan peran-peran yang ada dalam sistem. Setiap peran memiliki nama unik dan deskripsi.
* **Catatan BRD & Implementasi:**
    * **Autentikasi Berbasis Peran (RBAC):** Tabel ini secara langsung mendukung kebutuhan RBAC. Nama peran akan digunakan untuk mengidentifikasi dan mengontrol akses pengguna di sistem.

---

### `2025_06_05_000003_add_role_id_to_users_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            // Menambahkan kolom foreignId untuk role_id
            $table->foreignId('role_id')
                  ->nullable() // Peran bisa saja kosong sementara saat pendaftaran awal
                  ->constrained('roles') // Mengaitkan ke tabel 'roles'
                  ->onDelete('set null') // Jika peran dihapus, role_id di user menjadi NULL
                  ->after('password'); // Kolom ditempatkan setelah 'password'

            // Menambahkan indeks untuk performa pencarian dan penggabungan (join)
            $table->index('role_id');
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropForeign(['role_id']); // Menghapus foreign key constraint
            $table->dropColumn('role_id');    // Menghapus kolom role_id
            $table->dropIndex(['role_id']);   // Menghapus indeks
        });
    }
};
```

* **Penjelasan:** Migrasi ini menambahkan kolom `role_id` ke tabel `users`, menciptakan relasi Many-to-One (satu peran dapat dimiliki oleh banyak pengguna). Ini adalah implementasi dasar dari RBAC.
* **Catatan BRD & Implementasi:**
    * **RBAC:** Kolom ini memungkinkan setiap `user` memiliki peran tertentu, yang akan digunakan oleh Filament v3 untuk membatasi akses ke modul dan data.
    * **Klarifikasi BRD:** BRD menyebutkan "Otentikasi berbasis peran (RBAC) melalui panel admin Filament". Implementasi ini adalah fondasinya. Untuk kontrol akses yang lebih granular (misalnya, pengguna dengan peran 'pengajar' hanya boleh mengedit materi miliknya sendiri), Anda mungkin perlu paket pihak ketiga seperti Spatie Laravel Permission, yang akan menambah tabel `permissions` dan tabel pivot. Skema ini sudah cukup untuk membedakan hak akses di tingkat peran utama.

---

### `2025_06_05_000004_create_settings_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('settings', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('key')->unique(); // Kunci pengaturan unik (e.g., 'midtrans_server_key', 'whatsapp_api_url', 'default_salary_per_session') (VARCHAR 255)
            $table->text('value'); // Nilai pengaturan (TEXT)
            $table->text('description')->nullable(); // Deskripsi pengaturan (TEXT, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('settings');
    }
};
```

* **Penjelasan:** Tabel ini digunakan untuk menyimpan berbagai konfigurasi sistem yang dapat diubah melalui panel admin tanpa perlu modifikasi kode (misalnya, kunci API pihak ketiga, ambang batas kelulusan, dsb.).
* **Catatan BRD & Implementasi:**
    * **Asumsi Ketergantungan:** Mendukung penyimpanan kredensial untuk Midtrans API, WhatsApp API, dan SMTP Email yang disebutkan dalam BRD.

---

## 2. Modul Profil Perusahaan (Company Profile Module)

Modul ini bertanggung jawab atas informasi publik UEU Bootcamp yang ditampilkan di website.

---

### `2025_06_05_000005_create_pages_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('pages', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('title'); // Judul halaman (e.g., 'About Us', 'Home') (VARCHAR 255)
            $table->string('slug')->unique(); // Slug untuk URL yang ramah SEO (e.g., '/about-us') (VARCHAR 255)
            $table->longText('content'); // Isi halaman, bisa berupa HTML atau Markdown (LONGTEXT)
            $table->string('status')->default('draft'); // Status halaman: 'draft', 'published' (VARCHAR 255)
            $table->foreignId('created_by')->constrained('users')->onDelete('restrict'); // Siapa yang membuat/mengedit halaman
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('pages');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan konten untuk halaman statis seperti "Home", "About Us", dan "Contact Us". Kolom `slug` memungkinkan URL yang bersih dan mudah diingat.
* **Catatan BRD & Implementasi:**
    * **Company Profile:** Secara langsung mendukung fitur "Halaman Home, About Us, Contact Us". Admin dapat memperbarui konten informasi secara berkala.

---

### `2025_06_05_000006_create_blog_posts_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('blog_posts', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('title'); // Judul artikel blog (VARCHAR 255)
            $table->string('slug')->unique(); // Slug unik untuk URL artikel (VARCHAR 255)
            $table->longText('content'); // Isi artikel (LONGTEXT)
            $table->string('image')->nullable(); // Path ke gambar utama/thumbnail artikel (VARCHAR 255, bisa NULL)
            $table->timestamp('published_at')->nullable(); // Waktu publikasi artikel (TIMESTAMP, bisa NULL)
            $table->foreignId('author_id')->constrained('users')->onDelete('restrict'); // Penulis artikel (FK ke users)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('blog_posts');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan semua postingan blog, memungkinkan admin untuk mengelola artikel dan berita terkini.
* **Catatan BRD & Implementasi:**
    * **Company Profile (Blog):** Mendukung fitur "Blog" di mana pengunjung dapat membaca berbagai konten terkini.

---

### `2025_06_05_000007_create_contact_messages_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('contact_messages', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('name'); // Nama pengirim (VARCHAR 255)
            $table->string('email'); // Email pengirim (VARCHAR 255)
            $table->string('subject'); // Subjek pesan (VARCHAR 255)
            $table->text('message'); // Isi pesan (TEXT)
            $table->string('status')->default('new'); // Status pesan: 'new', 'read', 'archived', 'replied' (VARCHAR 255)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('contact_messages');
    }
};
```

* **Penjelasan:** Tabel ini berfungsi sebagai kotak masuk untuk pesan yang dikirim melalui formulir "Contact Us" di website.
* **Catatan BRD & Implementasi:**
    * **Company Profile (Contact Us):** Mendukung "Formulir kontak terintegrasi dengan sistem backend untuk menerima pesan dari pengguna."

---

## 3. Modul Human Resource Management (HRM)

Modul ini adalah inti dari pengelolaan sumber daya manusia, khususnya pengajar.

---

### `2025_06_05_000008_create_departments_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('departments', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('name')->unique(); // Nama departemen (e.g., 'Akademik', 'HR & Operasional', 'Research & Development') (VARCHAR 255)
            $table->text('description')->nullable(); // Deskripsi departemen (TEXT, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('departments');
    }
};
```

* **Penjelasan:** Tabel ini digunakan untuk mengkategorikan pengajar ke dalam departemen struktural UEU Bootcamp.
* **Catatan BRD & Implementasi:**
    * **HRM (Manajemen Departemen):** Mendukung pengelompokan pengajar ke dalam tim Akademik, HR, dan R&D.

---

### `2025_06_05_000009_create_branches_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('branches', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('name'); // Nama lokasi/cabang (e.g., 'Kampus A', 'Pusat Jakarta') (VARCHAR 255)
            $table->text('address'); // Alamat lengkap lokasi (TEXT)
            $table->string('phone_number')->nullable(); // Nomor telepon lokasi (VARCHAR 255, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('branches');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan informasi tentang lokasi pengajaran *offline* atau kantor cabang UEU Bootcamp.
* **Catatan BRD & Implementasi:**
    * **HRM (Manajemen Lokasi):** Mendukung penentuan lokasi *offline* pengajaran per pengajar.

---

### `2025_06_05_000010_create_teachers_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('teachers', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('user_id')->unique()->constrained('users')->onDelete('cascade'); // Relasi ke user account untuk login (FK, unik)
            $table->string('nik')->unique(); // Nomor Induk Kependudukan, harus unik (VARCHAR 255)
            $table->string('full_name'); // Nama lengkap pengajar (VARCHAR 255)
            $table->json('expertise')->nullable(); // Keahlian pengajar (JSON array, e.g., ["PHP", "Laravel", "Database"], bisa NULL)
            $table->boolean('is_active')->default(true); // Status aktif pengajar (BOOLEAN)
            $table->string('phone_number')->nullable(); // Nomor telepon pengajar (VARCHAR 255, bisa NULL)
            $table->text('address')->nullable(); // Alamat pengajar (TEXT, bisa NULL)
            $table->foreignId('department_id')->nullable()->constrained('departments')->onDelete('set null'); // Departemen pengajar (FK, bisa NULL)
            $table->foreignId('branch_id')->nullable()->constrained('branches')->onDelete('set null'); // Lokasi tugas utama (FK, bisa NULL)
            $table->integer('leave_quota')->default(5); // Kuota cuti per periode (sesuai BRD) (INTEGER)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('teachers');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan detail spesifik pengajar, yang berelasi One-to-One dengan tabel `users` (karena setiap pengajar memiliki akun login). Ini memisahkan data spesifik HRM dari data pengguna umum.
* **Catatan BRD & Implementasi:**
    * **HRM (Manajemen Data Pengajar):** Mendukung CRUD data pengajar, termasuk identitas, keahlian, status aktif. Riwayat dan aktivitas mengajar akan dicatat di tabel lain yang berelasi (`teacher_attendances`, `sessions`).
    * **Cuti dan Izin:** Kolom `leave_quota` secara langsung mendukung kebutuhan kuota cuti.

---

### `2025_06_05_000011_create_teacher_attendances_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('teacher_attendances', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('teacher_id')->constrained('teachers')->onDelete('cascade'); // Pengajar yang absen (FK)
            $table->foreignId('session_id')->constrained('sessions')->onDelete('cascade'); // Sesi yang dihadiri/tidak (FK)
            $table->date('attendance_date'); // Tanggal absensi (DATE)
            $table->time('clock_in_time')->nullable(); // Waktu clock-in (TIME, bisa NULL)
            $table->time('clock_out_time')->nullable(); // Waktu clock-out (TIME, bisa NULL)
            $table->enum('status', ['hadir', 'izin', 'sakit', 'alfa'])->default('alfa'); // Status absensi (ENUM)
            $table->text('notes')->nullable(); // Catatan tambahan (misal: alasan izin/sakit) (TEXT, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['teacher_id', 'session_id', 'attendance_date']); // Mencegah duplikasi absensi untuk sesi yang sama pada tanggal yang sama
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('teacher_attendances');
    }
};
```

* **Penjelasan:** Tabel ini mencatat kehadiran pengajar. Setiap entri mencatat absensi pengajar untuk sesi tertentu pada tanggal tertentu.
* **Catatan BRD & Implementasi:**
    * **HRM (Absensi Digital):** Mendukung "Clock-in/clock-out per sesi, rekap otomatis". `session_id` sangat penting di sini untuk mengaitkan absensi dengan sesi spesifik. Status `izin`/`sakit`/`alfa` juga didukung.

---

### `2025_06_05_000012_create_leave_requests_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('leave_requests', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('teacher_id')->constrained('teachers')->onDelete('cascade'); // Pengajar yang mengajukan cuti (FK)
            $table->date('start_date'); // Tanggal mulai cuti (DATE)
            $table->date('end_date'); // Tanggal akhir cuti (DATE)
            $table->integer('duration_days'); // Durasi cuti dalam hari, dihitung otomatis oleh aplikasi (INTEGER)
            $table->enum('type', ['cuti', 'izin', 'sakit']); // Tipe pengajuan cuti (ENUM)
            $table->text('reason'); // Alasan pengajuan cuti (TEXT)
            $table->string('proof_file')->nullable(); // Path ke file bukti (misal: surat dokter) (VARCHAR 255, bisa NULL)
            $table->enum('status', ['pending', 'approved', 'rejected'])->default('pending'); // Status pengajuan (ENUM)
            $table->foreignId('approved_by')->nullable()->constrained('users')->onDelete('set null'); // Admin yang menyetujui (FK, bisa NULL)
            $table->timestamp('approved_at')->nullable(); // Waktu persetujuan (TIMESTAMP, bisa NULL)
            $table->foreignId('replacement_teacher_id')->nullable()->constrained('teachers')->onDelete('set null'); // Pengajar pengganti yang ditunjuk (FK, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('leave_requests');
    }
};
```

* **Penjelasan:** Tabel ini mencatat setiap pengajuan cuti atau izin dari pengajar, termasuk status persetujuan dan informasi pengajar pengganti.
* **Catatan BRD & Implementasi:**
    * **HRM (Cuti dan Izin):** Mendukung pengajuan cuti digital, approval admin, dan sistem pengajar pengganti otomatis. "Pengajuan cuti minimal 3 hari sebelum tanggal" adalah aturan bisnis yang akan diimplementasikan di level aplikasi (validasi form).

---

### `2025_06_05_000013_create_salaries_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('salaries', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('teacher_id')->constrained('teachers')->onDelete('cascade'); // Pengajar yang menerima gaji (FK)
            $table->date('period_start'); // Tanggal awal periode gaji (DATE)
            $table->date('period_end');   // Tanggal akhir periode gaji (DATE)
            $table->integer('total_sessions_taught'); // Jumlah sesi mengajar di periode ini (INTEGER)
            $table->decimal('base_salary', 10, 2); // Gaji pokok (misal: 2.400.000,00) (DECIMAL)
            $table->decimal('overtime_pay', 10, 2)->default(0); // Pembayaran lembur (DECIMAL)
            $table->decimal('deductions', 10, 2)->default(0); // Total potongan (DECIMAL)
            $table->decimal('net_salary', 10, 2); // Gaji bersih (DECIMAL)
            $table->string('payslip_url')->nullable(); // Path ke slip gaji PDF (VARCHAR 255, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['teacher_id', 'period_start', 'period_end']); // Memastikan satu slip gaji per pengajar per periode
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('salaries');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan detail perhitungan gaji bulanan untuk setiap pengajar, termasuk gaji pokok, lembur, potongan, dan gaji bersih.
* **Catatan BRD & Implementasi:**
    * **HRM (Gaji):** Mendukung perhitungan gaji berbasis jumlah sesi, potongan, dan lembur. BRD menyebutkan aturan "Absen tanpa keterangan: 2,5% dari total gaji bulan itu per hari." Ini adalah **logika bisnis** yang akan diimplementasikan di aplikasi untuk menghitung `deductions`. Kolom `deductions` di sini akan menyimpan hasil perhitungan tersebut. ERD bisa mencakup bagaimana aturan potongan/lembur ini dikonfigurasi (misalnya, tabel `salary_rules` terpisah jika aturan sangat kompleks dan dinamis).

---

### `2025_06_05_000014_create_backup_teacher_assignments_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('backup_teacher_assignments', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('original_teacher_id')->constrained('teachers')->onDelete('cascade'); // Pengajar yang berhalangan (FK)
            $table->foreignId('backup_teacher_id')->constrained('teachers')->onDelete('cascade'); // Pengajar pengganti (FK)
            $table->foreignId('leave_request_id')->constrained('leave_requests')->onDelete('cascade'); // Tautan ke permintaan cuti (FK)
            $table->foreignId('session_id')->constrained('sessions')->onDelete('cascade'); // Sesi yang digantikan (FK)
            $table->text('notes')->nullable(); // Catatan tambahan untuk pengganti (TEXT, bisa NULL)
            $table->enum('status', ['pending', 'accepted', 'rejected', 'completed'])->default('pending'); // Status penugasan pengganti (ENUM)
            $table->timestamp('accepted_at')->nullable(); // Waktu pengajar pengganti menerima penugasan (TIMESTAMP, bisa NULL)
            $table->text('completion_report')->nullable(); // Laporan singkat dari pengajar pengganti setelah sesi selesai (TEXT, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('backup_teacher_assignments');
    }
};
```

* **Penjelasan:** Tabel ini melacak penugasan pengajar pengganti untuk sesi-sesi tertentu, terkait dengan permintaan cuti pengajar asli.
* **Catatan BRD & Implementasi:**
    * **HRM (Backup Teacher System):** Mendukung "Pemilihan pengajar cadangan, pemberian akses materi, dan notifikasi otomatis." Mekanisme pencarian pengajar cadangan dengan keahlian serupa akan berada di logika aplikasi, memanfaatkan kolom `expertise` di tabel `teachers`.

---

## 4. Modul Event Course

Modul ini menangani perencanaan dan pelaksanaan program bootcamp, termasuk penjadwalan dan evaluasi.

---

### `2025_06_05_000015_create_courses_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('courses', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->string('name'); // Nama program bootcamp (e.g., 'Fullstack Web Developer') (VARCHAR 255)
            $table->text('description')->nullable(); // Deskripsi program (TEXT, bisa NULL)
            $table->integer('duration_months'); // Durasi program dalam bulan (INTEGER)
            $table->integer('total_sessions'); // Total sesi dalam program ini (e.g., 72) (INTEGER)
            $table->decimal('price', 10, 2); // Biaya program (DECIMAL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('courses');
    }
};
```

* **Penjelasan:** Tabel ini mendefinisikan jenis-jenis program bootcamp yang ditawarkan oleh UEU Bootcamp.
* **Catatan BRD & Implementasi:**
    * **Event Course:** Menyediakan informasi dasar tentang program bootcamp.

---

### `2025_06_05_000016_create_batches_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('batches', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('course_id')->constrained('courses')->onDelete('cascade'); // Program yang diampu angkatan ini (FK)
            $table->string('name'); // Nama angkatan (e.g., "Batch 1 - Data Science") (VARCHAR 255)
            $table->date('start_date'); // Tanggal mulai angkatan (DATE)
            $table->date('end_date'); // Tanggal akhir angkatan (DATE)
            $table->enum('status', ['upcoming', 'active', 'completed', 'cancelled'])->default('upcoming'); // Status angkatan (ENUM)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('batches');
    }
};
```

* **Penjelasan:** Tabel ini merepresentasikan setiap "angkatan" atau implementasi spesifik dari sebuah program bootcamp.
* **Catatan BRD & Implementasi:**
    * **Event Course:** Mendukung pengelolaan perencanaan dan pelaksanaan program bootcamp selama enam bulan.

---

### `2025_06_05_000017_create_rp_ses_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('rps', function (Blueprint $table) { // RPS (Rencana Pembelajaran Semester)
            $table->id(); // Kunci utama otomatis
            $table->foreignId('batch_id')->constrained('batches')->onDelete('cascade'); // RPS untuk angkatan mana (FK)
            $table->string('title')->nullable(); // Judul RPS (misal: "RPS Angkatan 1 - Bulan 1") (VARCHAR 255, bisa NULL)
            $table->string('file_path'); // Path ke file RPS (PDF) (VARCHAR 255)
            $table->foreignId('uploaded_by')->constrained('users')->onDelete('restrict'); // Siapa yang mengunggah RPS (FK)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('rps');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan dokumen Rencana Pembelajaran Semester (RPS) untuk setiap angkatan bootcamp.
* **Catatan BRD & Implementasi:**
    * **Event Course:** Mendukung "Integrasi dengan RPS dan silabus" serta "RPS dapat diatur dan diunggah untuk tiap batch."

---

### `2025_06_05_000018_create_sessions_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('sessions', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('batch_id')->constrained('batches')->onDelete('cascade'); // Sesi milik angkatan mana (FK)
            $table->integer('session_number'); // Nomor sesi dalam angkatan (e.g., 1, 2, ..., 72) (INTEGER)
            $table->string('title'); // Judul sesi (e.g., 'Pengantar Pemrograman') (VARCHAR 255)
            $table->text('description')->nullable(); // Deskripsi sesi (TEXT, bisa NULL)
            $table->enum('type', ['online', 'offline', 'task', 'evaluation']); // Tipe sesi: daring, luring, tugas, evaluasi (ENUM)
            $table->dateTime('scheduled_at'); // Tanggal dan waktu jadwal sesi (DATETIME)
            $table->string('zoom_link')->nullable(); // Link Zoom untuk sesi daring (VARCHAR 255, bisa NULL)
            $table->foreignId('branch_id')->nullable()->constrained('branches')->onDelete('set null'); // Lokasi untuk sesi luring (FK, bisa NULL)
            $table->foreignId('teacher_id')->nullable()->constrained('teachers')->onDelete('set null'); // Pengajar yang ditugaskan (FK, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['batch_id', 'session_number']); // Memastikan nomor sesi unik per angkatan
            $table->index(['batch_id', 'scheduled_at']); // Indeks untuk performa pencarian jadwal
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('sessions');
    }
};
```

* **Penjelasan:** Tabel ini adalah jantung penjadwalan bootcamp, mendefinisikan setiap sesi pembelajaran.
* **Catatan BRD & Implementasi:**
    * **Event Course:** Mendukung "Penjadwalan otomatis 72 sesi selama 6 bulan." Kolom `session_number` dan `scheduled_at` sangat penting untuk ini.
    * **Format Pembelajaran Hybrid:** Kolom `type` (online/offline) dan `zoom_link`/`branch_id` mengakomodasi kebutuhan ini. BRD menyebut "2 teori dan 1 tugas per minggu", ini adalah logika penjadwalan yang akan diimplementasikan di aplikasi.
    * **Jadwal Siswa:** Tabel `sessions` ini secara langsung menjadi sumber data jadwal bagi siswa (melalui `student_enrollments` dan `batches`). Tidak diperlukan tabel migrasi terpisah untuk jadwal siswa.

---

### `2025_06_05_000019_create_evaluations_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('evaluations', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('batch_id')->constrained('batches')->onDelete('cascade'); // Evaluasi untuk angkatan mana (FK)
            $table->string('title'); // Judul evaluasi (e.g., "Evaluasi Tengah Angkatan X") (VARCHAR 255)
            $table->enum('type', ['mid_term', 'final']); // Tipe evaluasi: tengah atau akhir (ENUM)
            $table->dateTime('scheduled_at'); // Tanggal dan waktu jadwal evaluasi (DATETIME)
            $table->text('description')->nullable(); // Deskripsi evaluasi (TEXT, bisa NULL)
            $table->string('exam_link')->nullable(); // Link ke platform ujian eksternal atau modul kuis internal (VARCHAR 255, bisa NULL)
            $table->string('project_description_file')->nullable(); // Path ke file deskripsi proyek (untuk final) (VARCHAR 255, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('evaluations');
    }
};
```

* **Penjelasan:** Tabel ini mengelola detail evaluasi tengah dan akhir bootcamp.
* **Catatan BRD & Implementasi:**
    * **Event Course:** Mendukung "Evaluasi Tengah dan Akhir: Penjadwalan, pemberitahuan, dan pelaksanaan."

---

## 5. Modul Learning Management System (LMS)

Modul ini adalah platform pembelajaran digital utama bagi siswa.

---

### `2025_06_05_000020_create_modules_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('modules', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('session_id')->constrained('sessions')->onDelete('cascade'); // Modul terkait sesi mana (FK)
            $table->string('title'); // Judul modul (VARCHAR 255)
            $table->text('description')->nullable(); // Deskripsi modul (TEXT, bisa NULL)
            $table->string('file_path')->nullable(); // Path ke file dokumen (PDF, PPT, DOCX) (VARCHAR 255, bisa NULL)
            $table->string('video_url')->nullable(); // URL video pembelajaran (VARCHAR 255, bisa NULL)
            $table->string('external_link')->nullable(); // Link eksternal lainnya (VARCHAR 255, bisa NULL)
            $table->enum('type', ['document', 'video', 'link', 'other']); // Tipe konten modul (ENUM)
            $table->foreignId('uploaded_by')->constrained('users')->onDelete('restrict'); // Pengajar/admin yang upload (FK)

            // Kolom untuk proses review oleh Departemen Akademik
            $table->enum('review_status', ['pending', 'approved', 'rejected', 'revisions_requested'])->default('pending'); // Status review modul (ENUM)
            $table->foreignId('reviewed_by')->nullable()->constrained('users')->onDelete('set null'); // ID user yang melakukan review (FK, bisa NULL)
            $table->timestamp('reviewed_at')->nullable(); // Tanggal review dilakukan (TIMESTAMP, bisa NULL)
            $table->text('review_comments')->nullable(); // Komentar/catatan dari reviewer akademik (TEXT, bisa NULL)

            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('modules');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan semua materi pembelajaran digital yang diatur per sesi, dengan penambahan kolom untuk mendukung alur review oleh departemen akademik.
* **Catatan BRD & Implementasi:**
    * **LMS (Teaching & Modul Pembelajaran):** Mendukung "Upload materi pembelajaran (PDF, video, PPT), pembagian per sesi" dan "Materi dibuka bertahap sesuai minggu pembelajaran".
    * **Proses Review Akademik:** Penambahan kolom `review_status`, `reviewed_by`, `reviewed_at`, `review_comments` secara langsung menangani kebutuhan "disusun oleh pengajar dan diverifikasi oleh departemen akademik sebelum kelas dimulai." Logika aplikasi akan membatasi akses siswa hanya pada modul dengan `review_status = 'approved'`.

---

### `2025_06_05_000021_create_quizzes_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('quizzes', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('session_id')->constrained('sessions')->onDelete('cascade'); // Kuis terkait sesi mana (FK)
            $table->string('title'); // Judul kuis (VARCHAR 255)
            $table->text('description')->nullable(); // Deskripsi kuis (TEXT, bisa NULL)
            $table->integer('max_attempts')->default(1); // Jumlah maksimum percobaan kuis (INTEGER)
            $table->dateTime('due_date')->nullable(); // Batas waktu pengerjaan kuis (DATETIME, bisa NULL)
            $table->foreignId('created_by')->constrained('users')->onDelete('restrict'); // Pengajar/admin yang membuat kuis (FK)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('quizzes');
    }
};
```

* **Penjelasan:** Tabel ini mendefinisikan kuis-kuis yang tersedia, mengaitkannya dengan sesi pembelajaran.
* **Catatan BRD & Implementasi:**
    * **LMS (Quiz):** Mendukung fitur pembuatan kuis.

---

### `2025_06_05_000022_create_quiz_questions_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('quiz_questions', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('quiz_id')->constrained('quizzes')->onDelete('cascade'); // Pertanyaan milik kuis mana (FK)
            $table->text('question_text'); // Teks pertanyaan (TEXT)
            $table->enum('type', ['multiple_choice', 'essay', 'short_answer']); // Tipe pertanyaan (ENUM)
            $table->json('options')->nullable(); // Untuk pilihan ganda: [{"key": "A", "value": "Opsi A"}, {"key": "B", "value": "Opsi B"}] (JSON, bisa NULL)
            $table->text('correct_answer')->nullable(); // Kunci jawaban (untuk auto-grading) atau referensi (untuk esai) (TEXT, bisa NULL)
            $table->decimal('score_value', 5, 2); // Nilai untuk pertanyaan ini (DECIMAL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('quiz_questions');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan setiap pertanyaan untuk kuis, mendukung berbagai tipe pertanyaan.
* **Catatan BRD & Implementasi:**
    * **LMS (Quiz):** Mendukung "Pengajar dapat membuat soal dalam berbagai bentuk (pilihan ganda, uraian)." `options` sebagai JSON cukup fleksibel. "Skor otomatis dan manual" akan bergantung pada tipe pertanyaan dan logika aplikasi.

---

### `2025_06_05_000023_create_quiz_attempts_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('quiz_attempts', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('student_id')->constrained('users')->onDelete('cascade'); // Siswa yang mencoba kuis (FK ke users)
            $table->foreignId('quiz_id')->constrained('quizzes')->onDelete('cascade'); // Kuis yang dicoba (FK)
            $table->integer('attempt_number'); // Urutan percobaan (INTEGER)
            $table->timestamp('started_at')->nullable(); // Waktu mulai percobaan (TIMESTAMP, bisa NULL)
            $table->timestamp('submitted_at'); // Waktu pengajuan percobaan (TIMESTAMP)
            $table->decimal('score', 5, 2)->nullable(); // Nilai total upaya ini (DECIMAL, bisa NULL sampai dinilai)
            $table->text('feedback')->nullable(); // Umpan balik dari pengajar (TEXT, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['student_id', 'quiz_id', 'attempt_number']); // Memastikan unik per siswa, per kuis, per percobaan
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('quiz_attempts');
    }
};
```

* **Penjelasan:** Tabel ini mencatat setiap kali seorang siswa mencoba mengerjakan kuis.
* **Catatan BRD & Implementasi:**
    * **LMS (Quiz):** Menyediakan dasar untuk melacak percobaan kuis.

---

### `2025_06_05_000024_create_quiz_answers_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('quiz_answers', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('quiz_attempt_id')->constrained('quiz_attempts')->onDelete('cascade'); // Jawaban milik percobaan kuis mana (FK)
            $table->foreignId('question_id')->constrained('quiz_questions')->onDelete('cascade'); // Pertanyaan yang dijawab (FK)
            $table->longText('answer_text'); // Jawaban siswa (LONGTEXT)
            $table->boolean('is_correct')->nullable(); // True/False untuk auto-graded (BOOLEAN, bisa NULL)
            $table->decimal('graded_score', 5, 2)->nullable(); // Nilai yang diberikan untuk jawaban ini (DECIMAL, bisa NULL sampai dinilai)
            $table->foreignId('graded_by')->nullable()->constrained('users')->onDelete('set null'); // Pengajar yang menilai (FK, bisa NULL)
            $table->timestamp('graded_at')->nullable(); // Waktu penilaian dilakukan (TIMESTAMP, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['quiz_attempt_id', 'question_id']); // Memastikan satu jawaban per pertanyaan per percobaan
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('quiz_answers');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan jawaban spesifik siswa untuk setiap pertanyaan kuis, termasuk hasil penilaian.
* **Catatan BRD & Implementasi:**
    * **LMS (Quiz - Skor Otomatis & Manual):** `is_correct` untuk auto-grading dan `graded_score` untuk penilaian manual (esai/uraian).

---

### `2025_06_05_000025_create_tasks_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('tasks', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('session_id')->constrained('sessions')->onDelete('cascade'); // Tugas terkait sesi mana (FK)
            $table->string('title'); // Judul tugas (VARCHAR 255)
            $table->text('description')->nullable(); // Deskripsi tugas (TEXT, bisa NULL)
            $table->dateTime('due_date'); // Batas waktu pengumpulan tugas (DATETIME)
            $table->decimal('max_score', 5, 2); // Skor maksimum untuk tugas ini (DECIMAL)
            $table->foreignId('created_by')->constrained('users')->onDelete('restrict'); // Pengajar/admin yang membuat tugas (FK)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('tasks');
    }
};
```

* **Penjelasan:** Tabel ini mendefinisikan tugas-tugas yang diberikan kepada siswa, terkait dengan sesi pembelajaran.
* **Catatan BRD & Implementasi:**
    * **LMS (Pengelolaan Materi):** Mendukung fitur tugas yang disebutkan di BRD.

---

### `2025_06_05_000026_create_task_submissions_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('task_submissions', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('student_id')->constrained('users')->onDelete('cascade'); // Siswa yang submit tugas (FK ke users)
            $table->foreignId('task_id')->constrained('tasks')->onDelete('cascade'); // Tugas yang disubmit (FK)
            $table->string('file_path')->nullable(); // Path ke file tugas yang diunggah (VARCHAR 255, bisa NULL)
            $table->longText('submission_text')->nullable(); // Jawaban teks langsung jika ada (LONGTEXT, bisa NULL)
            $table->timestamp('submitted_at'); // Waktu pengumpulan tugas (TIMESTAMP)
            $table->decimal('score', 5, 2)->nullable(); // Nilai tugas (DECIMAL, bisa NULL sampai dinilai)
            $table->text('feedback')->nullable(); // Umpan balik dari pengajar (TEXT, bisa NULL)
            $table->foreignId('graded_by')->nullable()->constrained('users')->onDelete('set null'); // Pengajar yang menilai (FK, bisa NULL)
            $table->timestamp('graded_at')->nullable(); // Waktu penilaian (TIMESTAMP, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['student_id', 'task_id']); // Memastikan satu submission per siswa per tugas
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('task_submissions');
    }
};
```

* **Penjelasan:** Tabel ini mencatat pengumpulan tugas oleh siswa dan hasil penilaiannya.
* **Catatan BRD & Implementasi:**
    * **LMS (Pengelolaan Materi):** Mendukung "Pengumpulan tugas dan quiz setiap minggu."

---

### `2025_06_05_000027_create_student_attendances_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database\Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('student_attendances', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('student_id')->constrained('users')->onDelete('cascade'); // Siswa yang absen (FK ke users)
            $table->foreignId('session_id')->constrained('sessions')->onDelete('cascade'); // Sesi yang dihadiri/tidak (FK)
            $table->date('attendance_date'); // Tanggal absensi (DATE)
            $table->enum('status', ['hadir', 'izin', 'sakit', 'alfa'])->default('alfa'); // Status absensi (ENUM)
            $table->string('proof_screenshot_url')->nullable(); // URL bukti screenshot Zoom (untuk daring) (VARCHAR 255, bisa NULL)
            $table->text('reason')->nullable(); // Alasan izin/sakit (TEXT, bisa NULL)
            $table->foreignId('verified_by')->nullable()->constrained('users')->onDelete('set null'); // Pengajar yang memverifikasi absensi (FK, bisa NULL)
            $table->timestamp('verified_at')->nullable(); // Waktu verifikasi (TIMESTAMP, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['student_id', 'session_id']); // Memastikan satu absensi per siswa per sesi
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('student_attendances');
    }
};
```

* **Penjelasan:** Tabel ini mencatat kehadiran siswa untuk setiap sesi, mendukung mode *hybrid* (daring dan luring).
* **Catatan BRD & Implementasi:**
    * **LMS (Absensi Siswa Hybrid):** Mendukung "Absensi hybrid (daring dan luring)". `proof_screenshot_url` untuk daring dan `verified_by` untuk verifikasi manual. BRD menyebutkan "unggah tiga tangkapan layar Zoom (awal, tengah, akhir)". Saat ini, `proof_screenshot_url` hanya satu URL. Jika perlu detail per *screenshot*, kolom ini bisa diubah menjadi JSON array, atau dibuat tabel `student_attendance_proof_details` terpisah.

---

### `2025_06_05_000028_create_grades_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('grades', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('student_id')->constrained('users')->onDelete('cascade'); // Siswa (FK ke users)
            $table->foreignId('batch_id')->constrained('batches')->onDelete('cascade'); // Angkatan bootcamp (FK)
            $table->decimal('quiz_score_avg', 5, 2)->default(0); // Rata-rata nilai kuis (DECIMAL)
            $table->decimal('task_score_avg', 5, 2)->default(0); // Rata-rata nilai tugas (DECIMAL)
            $table->decimal('attendance_percentage', 5, 2)->default(0); // Persentase kehadiran (DECIMAL)
            $table->decimal('mid_term_score', 5, 2)->nullable(); // Nilai evaluasi tengah (DECIMAL, bisa NULL)
            $table->decimal('final_exam_score', 5, 2)->nullable(); // Nilai ujian akhir (DECIMAL, bisa NULL)
            $table->decimal('project_score', 5, 2)->nullable(); // Nilai proyek akhir (DECIMAL, bisa NULL)
            $table->decimal('final_grade', 5, 2)->default(0); // Nilai akhir akumulatif (DECIMAL)
            $table->boolean('is_passed')->default(false); // Status kelulusan (BOOLEAN)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['student_id', 'batch_id']); // Memastikan satu entri nilai akhir per siswa per angkatan
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('grades');
    }
};
```

* **Penjelasan:** Tabel ini menyimpan nilai akhir akumulatif siswa dari berbagai komponen penilaian.
* **Catatan BRD & Implementasi:**
    * **LMS (Nilai):** Mendukung "Perhitungan nilai akumulatif berdasarkan absensi, quiz, tugas, evaluasi tengah, dan proyek akhir." Kolom-kolom ini menyediakan tempat untuk menyimpan komponen-komponen tersebut. "Sistem menentukan status kelulusan peserta berdasarkan kombinasi nilai dan kehadiran minimal 85%." adalah logika aplikasi yang akan menggunakan data dari tabel ini.

---

### `2025_06_05_000029_create_certificates_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('certificates', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('student_id')->constrained('users')->onDelete('cascade'); // Siswa pemilik sertifikat (FK ke users)
            $table->foreignId('batch_id')->constrained('batches')->onDelete('cascade'); // Angkatan bootcamp (FK)
            $table->string('certificate_number')->unique(); // Nomor sertifikat unik (VARCHAR 255)
            $table->string('qr_code_value')->unique(); // Nilai unik untuk validasi QR Code (VARCHAR 255)
            $table->string('file_path'); // Path ke file sertifikat PDF (VARCHAR 255)
            $table->timestamp('issue_date'); // Tanggal sertifikat diterbitkan (TIMESTAMP)
            $table->boolean('is_issued')->default(false); // Status penerbitan sertifikat (BOOLEAN)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('certificates');
    }
};
```

* **Penjelasan:** Tabel ini mencatat sertifikat digital yang diterbitkan untuk siswa yang lulus.
* **Catatan BRD & Implementasi:**
    * **LMS (Sertifikat Otomatis):** Mendukung "Penerbitan sertifikat digital secara otomatis jika kriteria kelulusan terpenuhi." Kolom `qr_code_value` mendukung validasi sertifikat.

---

## 6. Modul Pendaftaran & Pembayaran (Student Enrollment & Payment Module)

Modul ini mengelola pendaftaran siswa dan proses pembayaran, terintegrasi dengan *payment gateway*.

---

### `2025_06_05_000030_create_student_enrollments_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('student_enrollments', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('student_id')->constrained('users')->onDelete('cascade'); // Siswa yang terdaftar (FK ke users)
            $table->foreignId('batch_id')->constrained('batches')->onDelete('cascade'); // Angkatan yang dipilih (FK)
            $table->timestamp('enrollment_date')->useCurrent(); // Tanggal pendaftaran (TIMESTAMP)
            $table->enum('status', ['pending_payment', 'active', 'completed', 'dropped', 'cancelled'])->default('pending_payment'); // Status pendaftaran (ENUM)
            $table->timestamps(); // `created_at` dan `updated_at`

            $table->unique(['student_id', 'batch_id']); // Memastikan satu siswa hanya bisa terdaftar satu kali di satu angkatan
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('student_enrollments');
    }
};
```

* **Penjelasan:** Tabel ini mencatat pendaftaran siswa ke angkatan bootcamp tertentu.
* **Catatan BRD & Implementasi:**
    * **Modul Student (Peserta):** Mendukung "Pendaftaran dan pemilihan course." Relasi ke `users` (sebagai siswa) dan `batches` (sebagai *course* yang dipilih).

---

### `2025_06_05_000031_create_payments_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate->Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('payments', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->foreignId('enrollment_id')->constrained('student_enrollments')->onDelete('cascade'); // Pembayaran untuk pendaftaran mana (FK)
            $table->string('midtrans_transaction_id')->unique()->nullable(); // ID transaksi dari Midtrans (unik, bisa NULL jika non-Midtrans) (VARCHAR 255)
            $table->string('order_id')->unique(); // ID pesanan dari sistem internal (VARCHAR 255)
            $table->decimal('amount', 10, 2); // Jumlah pembayaran (DECIMAL)
            $table->string('currency', 3)->default('IDR'); // Mata uang (VARCHAR 3)
            $table->string('payment_method')->nullable(); // Metode pembayaran (e.g., 'bank_transfer', 'gopay', 'va_bca') (VARCHAR 255, bisa NULL)
            $table->enum('status', ['pending', 'paid', 'failed', 'refunded', 'expired', 'challenge'])->default('pending'); // Status pembayaran (ENUM)
            $table->timestamp('transaction_date')->useCurrent(); // Waktu transaksi dibuat di sistem (TIMESTAMP)
            $table->timestamp('paid_at')->nullable(); // Waktu pembayaran terverifikasi oleh Midtrans/manual (TIMESTAMP, bisa NULL)
            $table->string('invoice_url')->nullable(); // Path ke invoice PDF (VARCHAR 255, bisa NULL)
            $table->json('raw_midtrans_response')->nullable(); // Untuk menyimpan respons lengkap dari webhook Midtrans (JSON, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('payments');
    }
};
```

* **Penjelasan:** Tabel ini mencatat semua transaksi pembayaran siswa, termasuk detail integrasi dengan Midtrans.
* **Catatan BRD & Implementasi:**
    * **Pembayaran Terintegrasi Midtrans:** Mendukung "Memfasilitasi pembayaran peserta secara digital" dan "Sistem mendeteksi status pembayaran secara otomatis" melalui `midtrans_transaction_id` dan `raw_midtrans_response`.
    * **Aktivasi LMS:** Setelah `status` menjadi `paid`, aplikasi akan otomatis mengaktifkan akses LMS peserta (logika aplikasi, bukan di database).
    * **Klarifikasi BRD:** BRD menyebutkan "Harus tersedia fallback berupa form unggah bukti manual jika koneksi dengan Midtrans bermasalah." Ini adalah alur kerja yang bisa diimplementasikan dengan menambahkan kolom `manual_proof_uploaded_at` dan `manual_verified_by` jika bukti manual perlu disimpan langsung di tabel ini. Untuk saat ini, diasumsikan verifikasi manual akan memperbarui status `payments` saja.

---

## 7. Sistem Notifikasi Otomatis (Automatic Notification System)

Modul ini mencatat semua notifikasi otomatis yang dikirim ke peserta dan pengajar.

---

### `2025_06_05_000032_create_notifications_log_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate->Database->Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('notifications_log', function (Blueprint $table) {
            $table->id(); // Kunci utama otomatis
            $table->morphs('notifiable'); // Relasi polimorfik ke model yang menerima notifikasi (e.g., User, Teacher, Student)
            $table->enum('channel', ['whatsapp', 'email']); // Saluran notifikasi (ENUM)
            $table->string('event_type'); // Jenis peristiwa yang memicu notifikasi (e.g., 'certificate_issued', 'payment_invoice', 'evaluation_reminder', 'leave_request_status_changed') (VARCHAR 255)
            $table->string('recipient_address'); // Nomor WhatsApp atau alamat email penerima (VARCHAR 255)
            $table->text('message'); // Isi pesan notifikasi (TEXT)
            $table->string('status')->default('sent'); // Status pengiriman: 'sent', 'failed', 'queued' (VARCHAR 255)
            $table->text('error_message')->nullable(); // Pesan error jika pengiriman gagal (TEXT, bisa NULL)
            $table->timestamps(); // `created_at` dan `updated_at`
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('notifications_log');
    }
};
```

* **Penjelasan:** Tabel ini berfungsi sebagai log untuk setiap notifikasi yang dikirim oleh sistem, memungkinkan pelacakan dan *troubleshooting*. Relasi polimorfik `notifiable` memungkinkan notifikasi terkait dengan berbagai model (misalnya, `User` sebagai siswa atau pengajar).
* **Catatan BRD & Implementasi:**
    * **Sistem Notifikasi Otomatis:** Mendukung "Fitur ini meningkatkan komunikasi dua arah dengan peserta dan pengajar." dan "Sistem secara otomatis mengirim notifikasi melalui API WhatsApp dan SMTP email." Tabel ini mencatat notifikasi *keluar*. Komunikasi "dua arah" akan memerlukan implementasi fitur *inbound messaging* dari API WhatsApp/Email, yang berada di luar cakupan skema database ini dan lebih pada logika aplikasi.

---

## Urutan Eksekusi Migrasi (Penting!)

Saat menjalankan migrasi, pastikan urutan ini diikuti karena adanya ketergantungan *foreign key*:

1.  `2025_06_05_000001_create_users_table.php`
2.  `2025_06_05_000002_create_roles_table.php`
3.  `2025_06_05_000003_add_role_id_to_users_table.php` (memodifikasi `users` dan bergantung pada `roles`)
4.  `2025_06_05_000004_create_settings_table.php`
5.  `2025_06_05_000005_create_pages_table.php` (bergantung pada `users` untuk `created_by`)
6.  `2025_06_05_000006_create_blog_posts_table.php` (bergantung pada `users` untuk `author_id`)
7.  `2025_06_05_000007_create_contact_messages_table.php`
8.  `2025_06_05_000008_create_departments_table.php`
9.  `2025_06_05_000009_create_branches_table.php`
10. `2025_06_05_000010_create_teachers_table.php` (bergantung pada `users`, `departments`, `branches`)
11. `2025_06_05_000015_create_courses_table.php`
12. `2025_06_05_000016_create_batches_table.php` (bergantung pada `courses`)
13. `2025_06_05_000017_create_rp_ses_table.php` (bergantung pada `batches`, `users`)
14. `2025_06_05_000018_create_sessions_table.php` (bergantung pada `batches`, `branches`, `teachers`)
15. `2025_06_05_000019_create_evaluations_table.php` (bergantung pada `batches`)
16. `2025_06_05_000011_create_teacher_attendances_table.php` (bergantung pada `teachers`, `sessions`)
17. `2025_06_05_000012_create_leave_requests_table.php` (bergantung pada `teachers`, `users`)
18. `2025_06_05_000013_create_salaries_table.php` (bergantung pada `teachers`)
19. `2025_06_05_000014_create_backup_teacher_assignments_table.php` (bergantung pada `teachers`, `leave_requests`, `sessions`)
20. `2025_06_05_000020_create_modules_table.php` (bergantung pada `sessions`, `users`)
21. `2025_06_05_000021_create_quizzes_table.php` (bergantung pada `sessions`, `users`)
22. `2025_06_05_000022_create_quiz_questions_table.php` (bergantung pada `quizzes`)
23. `2025_06_05_000023_create_quiz_attempts_table.php` (bergantung pada `users`, `quizzes`)
24. `2025_06_05_000024_create_quiz_answers_table.php` (bergantung pada `quiz_attempts`, `quiz_questions`, `users`)
25. `2025_06_05_000025_create_tasks_table.php` (bergantung pada `sessions`, `users`)
26. `2025_06_05_000026_create_task_submissions_table.php` (bergantung pada `users`, `tasks`)
27. `2025_06_05_000027_create_student_attendances_table.php` (bergantung pada `users`, `sessions`)
28. `2025_06_05_000028_create_grades_table.php` (bergantung pada `users`, `batches`)
29. `2025_06_05_000029_create_certificates_table.php` (bergantung pada `users`, `batches`)
30. `2025_06_05_000030_create_student_enrollments_table.php` (bergantung pada `users`, `batches`)
31. `2025_06_05_000031_create_payments_table.php` (bergantung pada `student_enrollments`)
32. `2025_06_05_000032_create_notifications_log_table.php` (bergantung pada `users` karena `morphs` bisa menunjuk ke `users`)
```