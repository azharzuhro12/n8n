# n8n — Admin Werehouse

Kumpulan workflow [n8n](https://n8n.io) untuk otomasi administrasi gudang (warehouse) melalui Telegram, dengan ekstraksi data berbantuan AI (Google Gemini) dan pencatatan ke Google Sheets.

## Daftar Workflow

| Folder | File | Fungsi |
| --- | --- | --- |
| `admin werhause/` | `Admin-Werehouse.json` | Pencatatan barang masuk/keluar dari pesan Telegram ke Google Sheets dan pembuatan laporan CSV |
| `minori/` | `minori_n8n_workflow.json` | Otomasi matching kandidat dan reminder WhatsApp/email (dry-run) dengan laporan harian ke manajemen |

## Admin Werehouse

Alur workflow:

1. **Telegram Trigger** menerima pesan masuk.
2. **If** menyaring pesan yang mengandung kata `laporan`.
3. **LLM Append data** (Google Gemini + Structured Output Parser) mengekstrak data dari pesan bebas dengan format:
   - `date` (YYYY-MM-DD)
   - `type` (`incoming` / `outgoing`)
   - `name` (nama barang/material)
   - `quantity` (angka)
   - `unit` (KG, liter, buah, sak, dll.)
   - `person` (penerima / penyetuju)
4. **Validate Date** memeriksa validitas tanggal.
5. Jika valid: baris ditambahkan ke **Google Sheets** lalu dikonfirmasi lewat Telegram.
   Jika tidak valid: pengirim menerima pesan format yang benar.
6. Untuk permintaan laporan: workflow membaca sheet, memfilter baris sesuai rentang tanggal, mengubahnya menjadi CSV, lalu mengirim file tersebut lewat Telegram.

Contoh hasil laporan ada di `admin werhause/laporan Gudang 2026-09-01 sampai 2026-09-30.csv` dengan kolom:

```
row_number,date,type,name,quantity,unit,person
```

## Minori

Workflow `minori/minori_n8n_workflow.json` berisi otomasi:

- Trigger manual (uji coba) dan jadwal harian 08:00 WIB.
- Matching engine untuk mencocokkan data kandidat.
- Reminder lewat WhatsApp (Fonnte) dan email kandidat.
- Laporan manajemen harian lewat email.

Node pengiriman sengaja dinonaktifkan (dry-run). Isi token Fonnte dan kredensial SMTP terlebih dahulu sebelum diaktifkan.

## Struktur Repo

```
n8n/
├── admin werhause/
│   ├── Admin-Werehouse.json
│   └── laporan Gudang 2026-09-01 sampai 2026-09-30.csv
├── minori/
│   ├── minori_n8n_workflow.json
│   └── graphviz.svg
├── nodes/
│   └── package.json
├── docker-compose.yml
├── .env.example
├── panduan setup n8n ngrog dan telegram.txt
└── README.md
```

## Prasyarat

- [Docker](https://docs.docker.com/engine/install/) dan Docker Compose
- Akun bot Telegram (buat lewat [@BotFather](https://t.me/BotFather))
- API key Google Gemini
- Google Sheets + kredensial OAuth2 Google
- (Opsional) [ngrok](https://ngrok.com) bila ingin mengekspos n8n ke internet

## Menjalankan n8n

### 1. Siapkan environment

```bash
cp .env.example .env
```

Isi `.env`, terutama:

- `N8N_ENCRYPTION_KEY` — buat dengan `openssl rand -base64 32`
- `WEBHOOK_URL` dan `N8N_EDITOR_BASE_URL` — URL publik n8n (mis. domain ngrok)

### 2. Jalankan container

```bash
docker compose up -d
```

n8n dapat diakses di `http://localhost:5678`.

> Catatan: `docker-compose.yml` memakai volume bernama `n8n_data`. Jika ingin memakai data n8n yang sudah ada di folder ini (bind mount), gunakan perintah `docker run` pada panduan atau ubah bagian `volumes` menjadi bind mount ke `./:/home/node/.n8n`.

### 3. Import workflow

1. Buka editor n8n.
2. Pilih **Workflows → Import from File**.
3. Pilih `admin werhause/Admin-Werehouse.json` atau `minori/minori_n8n_workflow.json`.
4. Simpan workflow.

## Kredensial yang Perlu Dibuat Ulang

ID kredensial di dalam file JSON hanya merujuk ke instance n8n asal dan **tidak akan berfungsi** setelah di-import. Buat ulang kredensial berikut di instance Anda, lalu hubungkan ke node terkait:

| Kredensial | Dipakai oleh |
| --- | --- |
| Telegram API | Telegram Trigger, node pengiriman pesan/dokumen |
| Google Gemini (PaLM) API | LLM Append data, LLM Report |
| Google Sheets OAuth2 | Get row(s) in sheet, Append row in sheet |
| Fonnte (HTTP Header Auth) | Kirim WhatsApp (Fonnte) |
| SMTP | Kirim Email Kandidat, Kirim Laporan ke Manajemen |

## Keamanan

File berikut adalah data runtime dan **tidak boleh** di-commit:

- `config` — berisi `encryptionKey` instance n8n. Dengan kunci ini, kredensial di `database.sqlite` bisa didekripsi.
- `database.sqlite`, `*-shm`, `*-wal` — database n8n beserta kredensial terenkripsi.
- `.env` — variabel environment dan kunci enkripsi.
- `n8nEventLog*.log` — log eksekusi.
- `storage/` — binary data hasil eksekusi.

Semua sudah tercantum di `.gitignore`. Pastikan `.env` dan `config` tidak pernah ter-commit, dan gunakan kunci enkripsi yang berbeda untuk setiap instance.

## Panduan Lengkap

Langkah setup n8n + ngrok + Telegram secara rinci ada di [panduan setup n8n ngrog dan telegram.txt](panduan%20setup%20n8n%20ngrog%20dan%20telegram.txt).
