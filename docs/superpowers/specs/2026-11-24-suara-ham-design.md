# Design Spec — Suara HAM

**Tanggal:** 2026-11-24  
**Topik:** Website Pengaduan Pelanggaran HAM & Aspirasi Demokrasi  
**Sumber aturan:** docs/md/RULES.md (Tugas kuliah: pengembangan & pengujian keamanan aplikasi berbasis Program Asta Cita menggunakan OWASP Top 10:2025)

## 1. Tujuan & Ruang Lingkup

Membuat website "Suara HAM" yang memungkinkan masyarakat mengadukan pelanggaran HAM, melihat katalog aspirasi demokrasi, dan melihat statistik/peta pelanggaran. Admin mengelola laporan, pengguna, dan hak akses.

Ketentuan minimum dari RULES.md:
- 2 pengguna (peran: Publik dan Admin)
- 3 fitur utama (kami sengaja mencakup 5 fitur agar uji keamanan lebih komprehensif)
- Database, autentikasi, pengaturan hak akses
- Pengujian di localhost terhadap seluruh 10 kategori OWASP Top 10:2025

## 2. Stack Teknologi

| Komponen | Pilihan | Alasan (satu baris) |
|----------|---------|---------------------|
| Backend framework | FastAPI | Ringan, async, bawaan OpenAPI/docs, cocok prototipe keamanan |
| Templating | Jinja2 | Render HTML sisi server, tanpa frontend terpisah |
| ORM / Database | SQLAlchemy + SQLite | Parameterized query bawaan (antisipasi injection), satu file database, mudah dijalankan localhost |
| Password hashing | bcrypt | Hash aman untuk autentikasi |
| Session | Cookie (server-side sederhana) | Mempertahankan status login antar request |
| Web server | uvicorn | Menjalankan FastAPI secara lokal |
| Styling | CSS eksternal (vanilla) | Tidak ada framework CSS bawaan AI, kontrol penuh atas warna dan tata letak |

## 3. Peran & Otorisasi

| Peran | Akses |
|-------|-------|
| Publik (belum login) | Halaman utama, katalog aspirasi, statistik/peta pelanggaran |
| Pelapor (login) | Semua publik + ajukan pengaduan, lihat laporan pribadi |
| Admin | Semua pelapor + lihat semua laporan, ubah status, kelola pengguna & hak akses |

Hanya 2 pengguna untuk memenuhi RULES.md: satu admin dan satu publik/pelapor. Tambahan pengguna bisa ditambahkan melalui manajemen pengguna admin.

## 4. Arsitektur (Opsi 2: Monolit dengan Service Layer)

Router hanya menangani HTTP (request/response). Logika bisnis dan CRUD berada di service layer. Router memanggil service, service memanggil database melalui SQLAlchemy.

### Alur data
Request → Router → Dependency autentikasi/otorisasi → Service (bisnis + DB) → Render template Jinja2 → Respons HTML

### Struktur file
```
app/
  main.py              # Inisialisasi FastAPI, daftar router, mount static/templates
  database.py          # Engine SQLAlchemy, session, Base
  models.py            # User, Report, Aspiration
  schemas.py           # Pydantic model validasi input
  auth.py              # bcrypt, session cookie, get_current_user, role check
  services/
    __init__.py
    user_service.py    # CRUD user, cek role, daftar pengguna
    report_service.py  # Ajukan, daftar, detail, ubah status laporan
    aspiration_service.py # CRUD katalog aspirasi
    stat_service.py    # Statistik dan data peta pelanggaran
  routers/
    public.py          # Halaman utama, katalog aspirasi, statistik/peta (tanpa login)
    reports.py         # Ajukan pengaduan (butuh login), daftar laporan pribadi
    admin.py           # Dashboard laporan semua, manajemen pengguna & hak akses (butuh admin)
  templates/
    base.html          # Template dasar Jinja2
    index.html         # Halaman utama
    katalog.html       # Katalog aspirasi
    statistik.html     # Statistik/peta pelanggaran
    laporan.html       # Ajukan & daftar laporan
    admin.html         # Dashboard admin & manajemen pengguna
    login.html         # Halaman login
  static/
    style.css          # CSS vanilla
data/
  suara_ham.db         # SQLite database file
```

## 5. Model Data

**User**
- id (PK), username (unique), password_hash, role (admin/publik), created_at

**Report**
- id (PK), judul, isi, lokasi, tanggal_kejadian, user_id (FK → User), status (pending/proses/selesai), created_at

**Aspiration**
- id (PK), judul, isi, kategori, created_at

## 6. Router & Endpoint

### public.py (tanpa autentikasi)
- GET / → halaman utama
- GET /aspirasi → katalog aspirasi
- GET /statistik → statistik & peta pelanggaran

### reports.py (butuh login)
- GET /laporan → daftar laporan pribadi
- GET /laporan/baru → form ajukan pengaduan
- POST /laporan/baru → simpan pengaduan
- GET /laporan/{id} → detail laporan pribadi

### admin.py (butuh role admin)
- GET /admin → dashboard laporan semua
- GET /admin/laporan/{id} → detail laporan (admin)
- POST /admin/laporan/{id}/status → ubah status laporan
- GET /admin/pengguna → daftar pengguna & hak akses
- POST /admin/pengguna → tambah/edit pengguna
- POST /admin/pengguna/{id}/hapus → hapus pengguna

### auth.py
- GET /login → form login
- POST /login → verifikasi & buat session
- POST /logout → hapus session

## 7. Penanganan Kesalahan

- 404: halaman tidak ditemukan
- 401: belum login (redirect ke /login)
- 403: tidak diizinkan (role tidak cukup)
- Flash message untuk kesalahan input (kata sandi salah, field kosong, dll.)
- Semua input divalidasi; query via SQLAlchemy parameterized (antisipasi injection)

## 8. Pengujian Keamanan (OWASP Top 10:2025)

Setiap kategori A01–A10 diuji manual dan melalui telaah kode/konfigurasi. Hasil dicatat dalam tabel laporan uji keamanan sesuai format RULES.md.

| Kode | Kategori | Contoh Pengujian |
|------|----------|------------------|
| A01 | Broken Access Control | Coba akses halaman admin sebagai publik |
| A02 | Security Misconfiguration | Cek header security, debug mode off |
| A03 | Software Supply Chain Failures | Cek dependensi terbaru, tidak ada paket tidak terdaftar |
| A04 | Cryptographic Failures | Password di-hash bcrypt, tidak tersimpan plainteks |
| A05 | Injection | Coba input karakter khusus, SQLAlchemy parameterized |
| A06 | Insecure Design | Telaah desain otorisasi per-role |
| A07 | Authentication Failures | Coba login password salah, session hijacking |
| A08 | Software or Data Integrity Failures | Validasi input, integritas data laporan |
| A09 | Security Logging and Alerting Failures | Log percobaan login gagal, akses admin |
| A10 | Mishandling of Exceptional Conditions | Error handling 404/403/500 tanpa expose stack trace |

## 9. Desain Visual (Antislop)

Tidak ada `DESIGN.md` maupun arahan brand dari user. Desain UI saat ini **draft tanpa arahan** (R-37). Dial default jujur: ENERGY 1 / RHYTHM 1 / MOTION 1. Tidak ada logo, avatar, atau statistik yang dibuat-buat; placeholder digunakan jika diperlukan.

Setiap teknik visual yang dipasang harus lolos uji tujuan: ditulis satu baris alasan mengapa teknik itu digunakan. Tidak ada em dash, tidak ada klaim buatan, tidak ada elemen interaktif yang mati, dan semua halaman harus memiliki estado kosong, memuat, dan error.

## 10. Keputusan Utama (Satu Baris Alasan Tiap Tiap)

- FastAPI + Jinja2: render HTML sisi server, cepat dibangun untuk tugas kuliah localhost.
- SQLAlchemy + SQLite: parameterized query bawaan, satu file database, tidak perlu server terpisah.
- Service layer (Opsi 2): memisahkan logika bisnis dari transport HTTP agar mudah diuji dan dipelihara.
- 2 pengguna (admin + publik): memenuhi ketentuan minimum RULES.md.
- 5 fitur: lebih dari minimum 3 agar uji keamanan OWASP lebih komprehensif.
- CSS vanilla: kontrol penuh atas tata letak, tidak ada framework CSS bawaan AI.

---

Spec self-review selesai: tidak ada placeholder TBD/TODO, tidak ada kontradiksi antar section, scope fokus untuk satu implementasi plan, dan tidak ada requirement ambigu.

Silakan review berkas spec ini. Berikan masukan jika ada perubahan sebelum kami lanjut ke rencana implementasi.