---
title: "WO-02: Migrasi Skema Canonical — tabel data core + referensi"
description: "Migrasi versioned untuk 9 tabel canonical (residents, mothers, birth_records, admissions, resident_status_history, facilities, facility_aliases, transfer_partners, resident_documents) + 5 tabel referensi (ref_religions, ref_blood_types, ref_delivery_methods, ref_attendant_types, ref_statuses) beserta seed nilai baku. Turunan §4 SPEC."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-02: Migrasi Skema Canonical

**SPEC acuan:** `SPEC.md` §4 (Definisi Tabel), §5 (resident_code), §6 (Pemetaan), §8 (Constraint), D-11, D-12
**Prasyarat:** WO-01

## Tujuan
Membuat seluruh tabel canonical + referensi lewat **migrasi versioned** (up & down), **tanpa `CREATE TABLE` di Go**.

## Cakupan Migrasi

### 0. Definisi field (sumber kebenaran, P-13)

Buat `plugins/resident/schema/fields.yaml` — daftar field canonical (nama, tipe, wajib, enum, label, kolom sumber Notion bila ada). Berkas ini menjadi acuan **migrasi, template (WO-08), dropdown referensi, dan validator (WO-06/09)**. Migrasi yang ditulis harus konsisten dengan berkas ini.

> Bila praktis, migrasi boleh di-*generate* dari `fields.yaml`; bila tidak, minimal ada uji/lint yang memastikan migrasi ⊇ daftar field di `fields.yaml`.

Pola file (meniru `plugins/endpoint/migrations/postgres/`):
```
plugins/resident/migrations/postgres/
├── 2026_10_27_120000_create_resident_reference_tables.up.sql
├── 2026_10_27_120000_create_resident_reference_tables.down.sql
├── 2026_10_27_120100_create_resident_core_tables.up.sql
└── 2026_10_27_120100_create_resident_core_tables.down.sql
```

> Referensi dibuat **lebih dulu** agar FK (mis. `residents.religion_code → ref_religions.code`) valid.

### Migrasi 1 — Tabel Referensi + seed
- `ref_religions(code PK, label)` → seed: `ISLAM`, `HINDU`, `KRISTEN`, `KATOLIK`, `BUDDHA`, `KONGHUCU`, `LAINNYA`
- `ref_blood_types(code PK, label, sort_order)` → `A+ A- B+ B- AB+ AB- O+ O-` (+ `O` bila disepakati; lihat catatan §6 SPEC)
- `ref_delivery_methods(code PK, label)` → `NORMAL`, `CAESAR`, `VAKUM`, `SENDIRI`, `LAINNYA`
- `ref_attendant_types(code PK, label)` → `DOKTER`, `BIDAN`, `DUKUN`, `SENDIRI`, `LAINNYA`
- `ref_statuses(code PK, label, is_terminal)` → `DIRAWAT`(false), `DIKEMBALIKAN`(true), `TRANSFER`(true), `MENINGGAL`(true)

### Migrasi 2 — Tabel Core
- `mothers` — `id UUID PK`, `full_name TEXT NOT NULL`, `nik VARCHAR(16) UNIQUE (partial WHERE nik IS NOT NULL)`, `phone`, `address`, `created_at`, `updated_at`
- `residents` — `id UUID PK`, `resident_code TEXT UNIQUE`, `branch_key TEXT NOT NULL`, `mother_id UUID NULL REFERENCES mothers(id)`, `full_name TEXT NOT NULL`, `gender TEXT NOT NULL CHECK (gender IN ('L','P'))`, `nik VARCHAR(16) UNIQUE partial`, `bpjs_no TEXT`, `birth_date DATE NOT NULL`, `birth_place TEXT`, `religion_code TEXT REFERENCES ref_religions(code)`, `blood_type_code TEXT REFERENCES ref_blood_types(code)`, `created_at`, `updated_at`
  - INDEX: `(branch_key)`, `(full_name)`, `(birth_date)`
- `birth_records` — `id UUID PK`, `resident_id UUID NOT NULL UNIQUE REFERENCES residents(id) ON DELETE CASCADE`, `birth_weight_gram INT`, `birth_length_cm INT`, `child_order INT`, `delivery_method_code TEXT REFERENCES ref_delivery_methods(code)`, `attendant_type_code TEXT REFERENCES ref_attendant_types(code)`, `facility_id UUID NULL REFERENCES facilities(id)`, `attendant_detail TEXT`
- `admissions` — `id UUID PK`, `resident_id UUID NOT NULL REFERENCES residents(id) ON DELETE CASCADE`, `admitted_at DATE NOT NULL`, `exited_at DATE`, `note TEXT`; INDEX `(resident_id, admitted_at DESC)`
- `resident_status_history` — `id UUID PK`, `resident_id UUID NOT NULL REFERENCES residents(id) ON DELETE CASCADE`, `status_code TEXT NOT NULL REFERENCES ref_statuses(code)`, `effective_at DATE NOT NULL`, `transfer_partner_id UUID NULL REFERENCES transfer_partners(id)`, `note TEXT`, `changed_by UUID NULL` (tanpa FK lintas-plugin; validasi di service), `created_at`; INDEX `(resident_id, effective_at DESC)`
- `facilities` — `id UUID PK`, `branch_key TEXT`, `name TEXT NOT NULL UNIQUE`, `facility_type TEXT CHECK (facility_type IN ('hospital','clinic','midwife_practice','foundation','other'))`, `is_active BOOLEAN NOT NULL DEFAULT TRUE`
- `facility_aliases` — `id UUID PK`, `facility_id UUID NOT NULL REFERENCES facilities(id) ON DELETE CASCADE`, `alias TEXT NOT NULL`; UNIQUE INDEX pada `LOWER(TRIM(alias))`
- `transfer_partners` — `id UUID PK`, `code TEXT NOT NULL UNIQUE`, `name TEXT NOT NULL`, `is_active BOOLEAN NOT NULL DEFAULT TRUE`; seed `JCH`
- `resident_documents` — `id UUID PK`, `resident_id UUID NOT NULL REFERENCES residents(id) ON DELETE CASCADE`, `doc_type TEXT NOT NULL CHECK (doc_type IN ('drive_folder','media','other'))`, `url TEXT NOT NULL`, `label TEXT`; INDEX `(resident_id, doc_type)`

> **Urutan pembuatan tabel** wajib memperhitungkan FK: `ref_*` & `transfer_partners` & `facilities` → `mothers` → `residents` → `birth_records`/`admissions`/`resident_status_history`/`resident_documents` → `facility_aliases`.

## Keputusan yang Dibutuhkan Sebelum WO Ini
| # | Pertanyaan | Default bila belum diputuskan |
|---|---|---|
| D-1 | Kolom wajib (`NOT NULL`) | Pakai wajib minimum: `full_name`, `gender`, `birth_date`, `branch_key` |
| D-8 | `resident_code` unik global/per cabang | **Global** (`UNIQUE(resident_code)`) |
| D-10 | PII at-rest | **Ditunda**: kolom PII dibuat `TEXT` biasa dulu, migrasi enkripsi terpisah. **Bila ingin aman sejak awal, tunda kolom `nik`/`bpjs_no`** |

> Bila D-10 belum diputuskan dan user ingin **fail-safe**, WO ini boleh **menunda** kolom `residents.nik`, `residents.bpjs_no`, `mothers.nik` ke WO terpisah. Catat pilihan yang diambil.

## Bukti yang Harus Dilaporkan
- `plugins/resident/schema/fields.yaml` ada; jumlah field tercatat.
- `./bin/console migrate up --plugin resident` pada DB bersih → semua tabel ada (`\dt` atau `information_schema`).
- `./bin/console migrate down --plugin resident` → semua tabel hilang.
- `migrate list` menampilkan 2 versi terpasang.
- Seed referensi terverifikasi: `SELECT code FROM ref_statuses` → 4 baris.
- `grep -rn "CREATE TABLE" plugins/resident/*.go` → **kosong**.
- `grep -rn "CREATE TABLE" --include=*.go internal/` → tidak ada tabel resident.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | 2 pasang migrasi (up/down) ada dan idempoten pada DB bersih. |
| AC-2 | 9 tabel canonical + 5 tabel referensi terbentuk. |
| AC-3 | Semua tabel referensi ter-seed nilai baku sesuai daftar di atas. |
| AC-4 | FK terpasang sesuai §4/§8 SPEC; `birth_records.resident_id` UNIQUE (1:1). |
| AC-5 | `resident_code` & `nik` UNIQUE (nik partial). |
| AC-6 | `facility_aliases` punya unique case/space-insensitive. |
| AC-7 | **Nol** `CREATE TABLE`/`INSERT ... seed` di kode Go. |
| AC-8 | `migrate down --plugin resident` membersihkan **semua** tabel yang dibuat (urutan aman FK). |
| AC-9 | `go build ./...`, `go vet`, `go test`, `gofmt -l` lulus. |
| AC-10 | Keputusan D-1/D-8/D-10 yang dipakai dicatat di laporan WO (atau eksplisit "ditunda"). |
| AC-11 | `plugins/resident/schema/fields.yaml` dibuat dan konsisten dengan kolom migrasi (ada uji/lint bila tidak di-generate). |
