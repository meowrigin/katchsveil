# Skema Migrasi Database UEU Bootcamp Terintegrasi

Diharapkan dengan dibentuknya dokumen ini, semoga panglima tempur kita yang bernama Iqbal akan mendapatkan pencerahan. Yang dimana pencerahan ini kelak akan membantunya disaat waktu yang benar benar dibutuhkan Iqbal dalam menyusun `ERD`.

---

Dokumen ini menyajikan seluruh skema migrasi *database* yang diperlukan untuk sistem terintegrasi UEU Bootcamp. Skema ini dirancang untuk mendukung operasional penuh modul Learning Management System (LMS) dan Human Resource Management (HRM), serta modul-modul pendukung lainnya.


-----

## Gambaran Umum Skema Database

Skema migrasi ini diorganisir ke dalam beberapa modul utama, mencerminkan struktur fungsional aplikasi:

1.  **Sistem Inti & Manajemen Pengguna:** Fondasi autentikasi, peran, dan konfigurasi umum sistem.
2.  **Modul Profil Perusahaan:** Mengelola informasi publik UEU Bootcamp untuk *website*.
3.  **Modul Human Resource Management (HRM):** Mengelola data dan aktivitas pengajar, termasuk absensi, cuti, dan penggajian.
4.  **Modul Event Course:** Mengelola perencanaan dan pelaksanaan program *bootcamp* serta penjadwalan.
5.  **Modul Learning Management System (LMS):** Platform pembelajaran digital untuk siswa, meliputi materi, kuis, tugas, dan penilaian.
6.  **Modul Pendaftaran & Pembayaran:** Mengelola proses pendaftaran siswa dan integrasi pembayaran.
7.  **Sistem Notifikasi Otomatis:** Mencatat semua notifikasi sistem yang dikirim.

-----

## Struktur Setiap Migrasi

Setiap *file* migrasi, yang diorganisir sesuai konvensi Laravel (`YYYY_MM_DD_HHMMSS_nama_migrasi.php`), akan mencakup detail berikut:

  * **Nama File Migrasi:** Identifikasi unik migrasi.
  * **Kode Migrasi PHP:** Struktur dasar Laravel untuk pembuatan atau modifikasi tabel.
  * **Penjelasan Tabel dan Kolom:** Detail fungsi setiap tabel, kolom, tipe data, dan relasi.
  * **Catatan BRD & Implementasi:** Keterangan mengenai bagaimana skema ini memenuhi kebutuhan BRD, serta potensi klarifikasi atau pertimbangan implementasi di tingkat aplikasi.

-----

## **Penting: Urutan Migrasi\!**

**Urutan eksekusi migrasi sangat krusial** karena adanya ketergantungan *foreign key*. Pastikan migrasi yang membuat tabel yang menjadi *foreign key* (kunci asing) dieksekusi terlebih dahulu. Untuk kemudahan, daftar urutan eksekusi yang direkomendasikan disertakan di akhir dokumen ini.

-----

## Modul-modul Migrasi

Berikut adalah detail skema migrasi untuk setiap modul:

-----

### 1\. Sistem Inti & Manajemen Pengguna (Core System & User Management)

Modul-modul ini adalah dasar dari seluruh aplikasi, menangani otentikasi, peran pengguna, dan konfigurasi umum.

  * `2025_06_05_000001_create_users_table.php`
  * `2025_06_05_000002_create_roles_table.php`
  * `2025_06_05_000003_add_role_id_to_users_table.php`
  * `2025_06_05_000004_create_settings_table.php`

-----

### 2\. Modul Profil Perusahaan (Company Profile Module)

Modul ini bertanggung jawab atas informasi publik UEU Bootcamp yang ditampilkan di *website*.

  * `2025_06_05_000005_create_pages_table.php`
  * `2025_06_05_000006_create_blog_posts_table.php`
  * `2025_06_05_000007_create_contact_messages_table.php`

-----

### 3\. Modul Human Resource Management (HRM)

Modul ini adalah inti dari pengelolaan sumber daya manusia, khususnya pengajar.

  * `2025_06_05_000008_create_departments_table.php`
  * `2025_06_05_000009_create_branches_table.php`
  * `2025_06_05_000010_create_teachers_table.php`
  * `2025_06_05_000011_create_teacher_attendances_table.php`
  * `2025_06_05_000012_create_leave_requests_table.php`
  * `2025_06_05_000013_create_salaries_table.php`
  * `2025_06_05_000014_create_backup_teacher_assignments_table.php`

-----

### 4\. Modul Event Course

Modul ini menangani perencanaan dan pelaksanaan program *bootcamp*, termasuk penjadwalan dan evaluasi.

  * `2025_06_05_000015_create_courses_table.php`
  * `2025_06_05_000016_create_batches_table.php`
  * `2025_06_05_000017_create_rp_ses_table.php`
  * `2025_06_05_000018_create_sessions_table.php`
  * `2025_06_05_000019_create_evaluations_table.php`

-----

### 5\. Modul Learning Management System (LMS)

Modul ini adalah platform pembelajaran digital utama bagi siswa.

  * `2025_06_05_000020_create_modules_table.php`
  * `2025_06_05_000021_create_quizzes_table.php`
  * `2025_06_05_000022_create_quiz_questions_table.php`
  * `2025_06_05_000023_create_quiz_attempts_table.php`
  * `2025_06_05_000024_create_quiz_answers_table.php`
  * `2025_06_05_000025_create_tasks_table.php`
  * `2025_06_05_000026_create_task_submissions_table.php`
  * `2025_06_05_000027_create_student_attendances_table.php`
  * `2025_06_05_000028_create_grades_table.php`
  * `2025_06_05_000029_create_certificates_table.php`

-----

### 6\. Modul Pendaftaran & Pembayaran (Student Enrollment & Payment Module)

Modul ini mengelola pendaftaran siswa dan proses pembayaran, terintegrasi dengan *payment gateway*.

  * `2025_06_05_000030_create_student_enrollments_table.php`
  * `2025_06_05_000031_create_payments_table.php`

-----

### 7\. Sistem Notifikasi Otomatis (Automatic Notification System)

Modul ini mencatat semua notifikasi otomatis yang dikirim ke peserta dan pengajar.

  * `2025_06_05_000032_create_notifications_log_table.php`

-----

## Urutan Eksekusi Migrasi yang Direkomendasikan

Agar proses migrasi berjalan lancar tanpa masalah *foreign key*, mohon ikuti urutan eksekusi berikut:

1.  `2025_06_05_000001_create_users_table.php`
2.  `2025_06_05_000002_create_roles_table.php`
3.  `2025_06_05_000003_add_role_id_to_users_table.php`
4.  `2025_06_05_000004_create_settings_table.php`
5.  `2025_06_05_000005_create_pages_table.php`
6.  `2025_06_05_000006_create_blog_posts_table.php`
7.  `2025_06_05_000007_create_contact_messages_table.php`
8.  `2025_06_05_000008_create_departments_table.php`
9.  `2025_06_05_000009_create_branches_table.php`
10. `2025_06_05_000010_create_teachers_table.php`
11. `2025_06_05_000015_create_courses_table.php`
12. `2025_06_05_000016_create_batches_table.php`
13. `2025_06_05_000017_create_rp_ses_table.php`
14. `2025_06_05_000018_create_sessions_table.php`
15. `2025_06_05_000019_create_evaluations_table.php`
16. `2025_06_05_000011_create_teacher_attendances_table.php`
17. `2025_06_05_000012_create_leave_requests_table.php`
18. `2025_06_05_000013_create_salaries_table.php`
19. `2025_06_05_000014_create_backup_teacher_assignments_table.php`
20. `2025_06_05_000020_create_modules_table.php`
21. `2025_06_05_000021_create_quizzes_table.php`
22. `2025_06_05_000022_create_quiz_questions_table.php`
23. `2025_06_05_000023_create_quiz_attempts_table.php`
24. `2025_06_05_000024_create_quiz_answers_table.php`
25. `2025_06_05_000025_create_tasks_table.php`
26. `2025_06_05_000026_create_task_submissions_table.php`
27. `2025_06_05_000027_create_student_attendances_table.php`
28. `2025_06_05_000028_create_grades_table.php`
29. `2025_06_05_000029_create_certificates_table.php`
30. `2025_06_05_000030_create_student_enrollments_table.php`
31. `2025_06_05_000031_create_payments_table.php`
32. `2025_06_05_000032_create_notifications_log_table.php`

-----
