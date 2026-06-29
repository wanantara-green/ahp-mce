# Stack AHP-MCE — Pembobotan Zonasi

Stack lengkap untuk pengumpulan penilaian ahli (pairwise AHP), perhitungan bobot
deterministik, dan pelaporan agregat — siap dijalankan dengan satu perintah Docker Compose.

```
┌──────────────┐     /api/ proxy    ┌──────────────┐     psql     ┌────────────┐
│  frontend    │ ─────────────────▶│   backend    │ ───────────▶ │     db     │
│ nginx +      │                    │ Django + DRF │              │ PostgreSQL │
│ Tailwind UI  │ ◀─────────────────│ kobo_mce AHP │ ◀─────────── │            │
└──────────────┘                    └──────────────┘              └────────────┘
   :8080                                :8000 (internal)              :5432 (internal)
```

## Menjalankan

### Pengembangan (live-reload, default)

`docker-compose.override.yml` dimuat otomatis: kode di-mount (frontend & backend),
`DJANGO_DEBUG=1`, backend pakai runserver auto-reload.

```bash
cp .env.example .env        # sesuaikan password & secret
docker compose up --build
```

→ Edit HTML/JS/CSS frontend lalu **refresh browser**; edit `.py` backend → **auto-reload**. Tanpa rebuild.

### Produksi

Abaikan override; jalankan file base saja (gunicorn, kode ter-bake ke image):

```bash
docker compose -f docker-compose.yml up -d --build
```

### Akses

- **http://localhost:8080** — antarmuka penilaian ahli (form satu halaman, 28 perbandingan pairwise)
- **http://localhost:8080/hasil.html** — dashboard hasil agregat
- **http://localhost:8080/admin/** — Django admin (tema Claude/Jazzmin; buat superuser dulu)

Membuat superuser admin:
```bash
docker compose exec backend python manage.py createsuperuser
```

> **Data awal:** saat instalasi pertama (database kosong), migrasi `0002_seed_initial_experts`
> otomatis memuat **15 penilaian pakar simulasi** (fixture `kobo_mce/fixtures/initial_experts.json`).
> Migrasi ini tidak menimpa data bila tabel sudah berisi.

## Layanan

| Layanan | Basis | Peran |
|---|---|---|
| `db` | postgres:16-alpine | Penyimpanan respons ahli & nilai pairwise |
| `backend` | python:3.12 + Django/DRF | API AHP: submit, validasi CR, agregasi, narasi AI |
| `frontend` | nginx:1.27-alpine | UI statis Tailwind + proksi `/api/` ke backend |

## Endpoint API

| Metode | Path | Fungsi |
|---|---|---|
| POST | `/api/submit/` | Simpan penilaian satu ahli (+ validasi CR otomatis) |
| GET | `/api/experts/` | Daftar ahli & jumlah per tipologi |
| GET | `/api/weights/` | Bobot global 16 indikator + perbandingan tipologi |
| POST | `/api/validate/` | Cek CR satu set pairwise tanpa menyimpan (feedback real-time) |
| POST | `/api/narrative/` | Narasi interpretasi via Claude API (opsional) |

## Hierarki penilaian

5 kriteria utama (konstruk) dan **16 indikator**, total **28 perbandingan** berpasangan per ahli:

| Blok | Kriteria | Indikator | Pasangan |
|---|---|---|---|
| konstruk | Antar kriteria utama | K1–K5 (5) | 10 |
| k1 | Kesesuaian Lahan | Kakao, Padi, Lainnya (Jagung/Sagu/Cengkeh) (3) | 3 |
| k2 | Daya Dukung Lingkungan | Lahan, Air, Jasa Ekosistem (3) | 3 |
| k3 | Risiko Iklim & Bencana | Banjir/Longsor, Kekeringan, Hidrometeorologi (3) | 3 |
| k4 | Nilai Konservasi | ABKT, Preservasi, Fungsi Hidrologi (3) | 3 |
| k5 | Faktor Sosial-Ekonomi | Permukiman, Infrastruktur, Diversifikasi, Aktivitas Ekonomi (4) | 6 |

## Alur pengguna

1. Ahli mengisi identitas + memilih tipologi (4 kategori).
2. Form **satu halaman** menampilkan seluruh 28 perbandingan sekaligus, dikelompokkan per blok.
3. Menggeser penanda bipolar (skala Saaty −9…+9); **CR tiap kelompok diperbarui langsung** (badge hijau = konsisten, merah = perlu ditinjau).
4. Kirim → tersimpan di PostgreSQL, otomatis ditandai konsisten/tidak. Respons dengan CR > 0,10 dikecualikan dari agregat.
5. Dashboard `hasil.html` menampilkan bobot global teragregasi + perbandingan antar tipologi.

## Konvensi nilai pairwise (penting)

Geseran slider dikonversi ke nilai bertanda yang dipahami backend:
- Slider ke **kiri** → elemen kiri (i) lebih penting → nilai **positif** (skala Saaty)
- Slider ke **kanan** → elemen kanan (j) lebih penting → nilai **negatif** (→ 1/skala)
- Tengah → 1 (sama penting)

Konvensi ini sudah diverifikasi konsisten ujung-ke-ujung dengan `signed_to_ratio()`
di `kobo_mce/weighting/ahp.py`.

## Lapisan AI (opsional)

Isi `ANTHROPIC_API_KEY` di `.env` untuk mengaktifkan endpoint `/api/narrative/`.
Bila kosong, endpoint mengembalikan 503 dan fitur narasi nonaktif — bagian
perhitungan bobot tetap berjalan penuh tanpa API key (bobot bersifat deterministik;
AI hanya untuk interpretasi naratif, tidak pernah mengubah angka).

## Dev vs Produksi

| | Dev (`docker compose up`) | Produksi (`-f docker-compose.yml`) |
|---|---|---|
| Override dimuat | Ya (otomatis) | Tidak |
| Kode | di-mount (live-reload) | ter-bake ke image |
| Backend | runserver + `DJANGO_DEBUG=1` | gunicorn 3 worker + `DJANGO_DEBUG=0` |

Untuk produksi: ganti `DJANGO_SECRET_KEY` & `POSTGRES_PASSWORD` dengan nilai acak.
Frontend dan API satu origin (proksi nginx), jadi tidak ada masalah CORS.

## Struktur direktori

```
ahp-mce/
├── docker-compose.yml             # base (produksi)
├── docker-compose.override.yml    # override dev (live-reload, auto dimuat)
├── .env.example
├── backend/
│   ├── Dockerfile  ·  entrypoint.sh  ·  requirements.txt  ·  manage.py
│   ├── ahp_mce/            # proyek Django (settings, urls, wsgi)
│   ├── templates/admin/    # override template admin (submit_line.html)
│   └── kobo_mce/           # app AHP
│       ├── weighting/      # ahp.py, aggregate.py  (inti deterministik)
│       ├── services/       # mce_pipeline.py
│       ├── models.py  ·  parser.py  ·  serializers.py  ·  views.py
│       ├── api_urls.py  ·  admin.py  ·  ai_layer.py
│       ├── static/admin_extra/   # tema Claude (admin_extra.css) + favicon.svg
│       ├── fixtures/             # initial_experts.json (seed 15 pakar)
│       └── migrations/           # 0001_initial, 0002_seed_initial_experts
└── frontend/
    ├── Dockerfile
    ├── nginx/default.conf
    └── static/
        ├── index.html      # form penilaian satu halaman (28 perbandingan)
        ├── hasil.html      # dashboard hasil agregat
        ├── config.js       # struktur hierarki (cermin backend)
        ├── app.js          # logika interaksi + validasi CR live
        └── favicon.svg     # ikon brand (sama dengan admin)
```
