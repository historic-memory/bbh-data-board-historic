---
title: "WO-02: Migrasi Skema Canonical — tabel data core + referensi"
description: "Migrasi versioned untuk 8 tabel canonical (residents, mothers, birth_records, admissions, resident_status_history, facilities, facility_aliases, resident_documents) + 5 tabel referensi (ref_religions, ref_blood_types, ref_delivery_methods, ref_attendant_types, ref_statuses) beserta seed nilai baku. Turunan §4 SPEC. Tidak ada tabel transfer_partners (D-6: tujuan transfer cukup di note)."
status: complete
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
├── 2026_10_27_120100_create_resident_core_tables.up.sql
└── 2026_10_27_120100_create_resident_core_tables.down.sql
```

> **Tidak ada migrasi tabel referensi** (D-3): nilai kategorikal = **enum statis** (konstanta Go + `CHECK` constraint), label lewat i18n.

### Migrasi — Tabel Core (8 tabel)
- `mothers` — `id UUID PK`, `full_name TEXT NOT NULL`, `nik VARCHAR(16) UNIQUE (partial WHERE nik IS NOT NULL)`, `phone`, `address`, `created_at`, `updated_at`
- `facilities` — `id UUID PK`, `branch_key TEXT`, `name TEXT NOT NULL UNIQUE`, `facility_type TEXT CHECK (facility_type IN ('hospital','clinic','midwife_practice','foundation','other'))`, `is_active BOOLEAN NOT NULL DEFAULT TRUE`
- `facility_aliases` — `id UUID PK`, `facility_id UUID NOT NULL REFERENCES facilities(id) ON DELETE CASCADE`, `alias TEXT NOT NULL`; UNIQUE INDEX pada `LOWER(TRIM(alias))`
- `residents` — `id UUID PK`, `resident_code TEXT UNIQUE`, `branch_key TEXT NOT NULL`, `mother_id UUID NULL REFERENCES mothers(id)`, `full_name TEXT NOT NULL`, `gender TEXT NOT NULL CHECK (gender IN ('L','P'))`, `nik VARCHAR(16) UNIQUE partial`, `bpjs_no TEXT`, `birth_date DATE NOT NULL`, `birth_place TEXT`, `religion_code TEXT CHECK (religion_code IN ('ISLAM','HINDU','KRISTEN','KATOLIK','BUDDHA','KONGHUCU','LAINNYA'))`, `blood_type_code TEXT CHECK (blood_type_code IN ('A+','A-','B+','B-','AB+','AB-','O+','O-','O'))`, `created_at`, `updated_at`
  - INDEX: `(branch_key)`, `(full_name)`, `(birth_date)`
- `birth_records` — `id UUID PK`, `resident_id UUID NOT NULL UNIQUE REFERENCES residents(id) ON DELETE CASCADE`, `birth_weight_gram INT`, `birth_length_cm INT`, `child_order INT`, `delivery_method_code TEXT CHECK (delivery_method_code IN ('NORMAL','CAESAR','VAKUM','SENDIRI','LAINNYA'))`, `attendant_type_code TEXT CHECK (attendant_type_code IN ('DOKTER','BIDAN','DUKUN','SENDIRI','LAINNYA'))`, `facility_id UUID NULL REFERENCES facilities(id)`, `attendant_detail TEXT` (teks nama saja — D-5)
- `admissions` — `id UUID PK`, `resident_id UUID NOT NULL REFERENCES residents(id) ON DELETE CASCADE`, `admitted_at DATE NOT NULL`, `exited_at DATE`, `note TEXT`; INDEX `(resident_id, admitted_at DESC)`
- `resident_status_history` — `id UUID PK`, `resident_id UUID NOT NULL REFERENCES residents(id) ON DELETE CASCADE`, `status_code TEXT NOT NULL CHECK (status_code IN ('DIRAWAT','DIKEMBALIKAN','TRANSFER','MENINGGAL'))`, `effective_at DATE NOT NULL`, `note TEXT` (menampung tujuan transfer, mis. "transfer ke JCH" — D-6), `changed_by UUID NULL` (tanpa FK lintas-plugin; validasi di service), `created_at`; INDEX `(resident_id, effective_at DESC)`
- `resident_documents` — `id UUID PK`, `resident_id UUID NOT NULL REFERENCES residents(id) ON DELETE CASCADE`, `doc_type TEXT NOT NULL CHECK (doc_type IN ('drive_folder','media','other'))`, `url TEXT NOT NULL`, `label TEXT`; INDEX `(resident_id, doc_type)`

> **TIDAK ADA** tabel `transfer_partners` (D-6), **TIDAK ADA** tabel `ref_*` (D-3).

> **Urutan pembuatan tabel** memperhitungkan FK: `facilities` -> `mothers` -> `residents` -> `birth_records`/`admissions`/`resident_status_history`/`resident_documents` -> `facility_aliases`.

## Keputusan yang Sudah Dikunci
| # | Keputusan | Implikasi migrasi |
|---|---|---|
| D-1 | Satu standar: field wajib minimum ditetapkan | `NOT NULL` pada: `residents.full_name`, `residents.gender`, `residents.birth_date`, `residents.branch_key`. Usulan tambahan yang perlu dikonfirmasi: `admissions.admitted_at`, status awal. |
| D-8 | `resident_code` UNIQUE **global** | `UNIQUE(resident_code)`; kosong disimpan `NULL` (bukan `''`) agar tidak bentrok |
| D-10 | PII **plaintext** | `nik`, `bpjs_no` bertipe `VARCHAR(16)`/`TEXT` biasa; **tidak ada** enkripsi di WO ini |
| D-9 | Dokumen = link | `resident_documents.url TEXT NOT NULL`, tanpa integrasi storage |
| D-3 | Enum **statis** | Kolom enum = `TEXT` + `CHECK`; **tanpa** tabel `ref_*` |
| D-2 | `Keterangan` = tempat persalinan | Didukung `facilities` + `facility_aliases` (tabel tetap ada) |
| D-5 | `attendant_detail` | `TEXT` nama saja, tanpa tabel tenaga medis |
| D-6 | Transfer | Tanpa tabel `transfer_partners`; cukup `note` |

## Bukti yang Harus Dilaporkan
- `plugins/resident/schema/fields.yaml` ada; jumlah field tercatat.
- `./bin/console migrate up --plugin resident` pada DB bersih → semua tabel ada (`\dt` atau `information_schema`).
- `./bin/console migrate down --plugin resident` → semua tabel hilang.
- `migrate list` menampilkan versi terpasang (1 migrasi resident).
- Enum statis terbukti: `INSERT` nilai enum di luar daftar -> ditolak `CHECK` constraint.
- `grep -rn "CREATE TABLE" plugins/resident/*.go` -> **kosong**.
- `grep -rn "CREATE TABLE" --include=*.go internal/` → tidak ada tabel resident.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | 1 pasang migrasi (up/down) ada dan idempoten pada DB bersih. |
| AC-2 | 8 tabel canonical terbentuk; **tidak ada** tabel `ref_*`/`transfer_partners`. |
| AC-3 | Semua kolom enum memakai `CHECK` constraint sesuai daftar §4.10 SPEC. |
| AC-4 | FK terpasang sesuai §4/§8 SPEC; `birth_records.resident_id` UNIQUE (1:1). |
| AC-5 | `resident_code` & `nik` UNIQUE (nik partial). |
| AC-6 | `facility_aliases` punya unique case/space-insensitive. |
| AC-7 | **Nol** `CREATE TABLE`/`INSERT ... seed` di kode Go. |
| AC-8 | `migrate down --plugin resident` membersihkan **semua** tabel yang dibuat (urutan aman FK). |
| AC-9 | `go build ./...`, `go vet`, `go test`, `gofmt -l` lulus. |
| AC-10 | Keputusan D-1/D-8/D-10 yang dipakai dicatat di laporan WO. |
| AC-11 | `plugins/resident/schema/fields.yaml` dibuat dan konsisten dengan kolom migrasi (ada uji/lint bila tidak di-generate). |
| AC-12 | `resident_code` UNIQUE global; nilai kosong tersimpan `NULL`, bukan string kosong. |

## Status Eksekusi (complete)
**Status: complete** — commit kode `7c32174` (`feat(resident): WO-02 canonical schema migration + fields.yaml`). Tidak ada push.

Artefak:
- `plugins/resident/schema/fields.yaml` — **64 field** pada 8 tabel + 8 grup enum + peta kolom template (P-13).
- `plugins/resident/schema/schema.go` — `go:embed` loader (`Load/MustLoad/LookupField/TableNames/Raw`).
- `plugins/resident/migrations/postgres/2026_10_27_120100_create_resident_core_tables.{up,down}.sql`.

| # | AC | Hasil |
|---|---|---|
| AC-1 | 1 pasang migrasi, idempoten | ✅ `migrate up` → current=1; `down` → current=0; `up` ulang OK |
| AC-2 | 8 tabel; tanpa `ref_*`/`transfer_partners` | ✅ `information_schema` = 8 tabel; query `ref_*`/`transfer_partners`/`guardians` = 0 baris |
| AC-3 | Enum via `CHECK` | ✅ `INSERT gender='X'` & `religion_code='PASTAFARIAN'` → ditolak `check constraint` |
| AC-4 | FK + `birth_records.resident_id` UNIQUE | ✅ FK sesuai §4/§8; `resident_id ... UNIQUE` |
| AC-5 | `resident_code` & `nik` UNIQUE (nik partial) | ✅ dup `BBH-26-P-001` → `unique constraint` |
| AC-6 | alias unik case/space-insensitive | ✅ `'RS. Premagana'` vs `'  rs. premagana  '` → ditolak (`uq_facility_aliases_alias`) |
| AC-7 | Nol DDL/seed di Go | ✅ tidak ada `CREATE TABLE`/`INSERT` DDL di `.go` (hanya regex di `schema_test.go`) |
| AC-8 | `down` bersihkan semua | ✅ 8 tabel → 0 setelah down |
| AC-9 | build/vet/test/gofmt | ✅ lulus di container `app` |
| AC-10 | Keputusan dicatat | ✅ lihat catatan di bawah |
| AC-11 | fields.yaml konsisten migrasi | ✅ `schema_test.go` (`TestFieldsMatchMigration`) lulus |
| AC-12 | UNIQUE global; kosong = NULL | ✅ 2 baris `resident_code IS NULL` diterima; duplikat ditolak |

**Penyimpangan sadar (dicatat):** PK memakai `TEXT` UUIDv7 (app-generated) — **bukan** literal `UUID` di draft WO — agar konsisten dengan plugin `auth` (`TEXT PRIMARY KEY`, tanpa `gen_random_uuid()`/ekstensi). `changed_by` juga `TEXT` (menyamai `admins.id`).

**D-1/D-8/D-10 yang dipakai:** `NOT NULL` pada `residents.full_name/gender/birth_date/branch_key` (D-1); `resident_code UNIQUE` dgn `NULL` (D-8); `nik`/`bpjs_no` plaintext tanpa enkripsi (D-10).
