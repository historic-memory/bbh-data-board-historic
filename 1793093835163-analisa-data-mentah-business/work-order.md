---
title: "Work Order (Master Index): Plugin Resident — Data Core BBH"
description: "Rencana implementasi plugin `resident`: skema canonical data bayi/penghuni BBH lintas cabang, API admin, CLI, dan importer CSV. Diturunkan dari SPEC.md (ERD data core) topic ini."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---
# Work Order (Master Index): Plugin Resident — Data Core BBH

## Referensi
- SPEC: `SPEC.md` di topic ini (P-1..P-12, D-1..D-13, AC-1..AC-12)
- Data mentah: `Data Bayi Yayasan 1b590f1a8a518093ac5bd1d366590b33_all.csv` (50 baris, 23 kolom)
- Topic AI-ETL (di luar lingkup, terpisah): `.historic/1793096602507-integrasi-llm-ai-etl-copilotkit-jobmanager-sandbox-acp-agent/`

## Keputusan yang Sudah Dikunci (dari SPEC)
| # | Keputusan |
|---|---|
| D-11 | Identitas cabang memakai `endpoints.key` — **tidak ada tabel `branches`** |
| D-12 | Tabel referensi `ref_*` tinggal di dalam plugin `resident` dulu |
| D-13 | Nama plugin: **`plugins/resident`** |

## Keputusan yang Masih Terbuka (tidak menghalangi WO-01..WO-02)
| # | Pertanyaan | Dampak bila belum diputuskan |
|---|---|---|
| D-1 | Titik tengah = super-set + wajib minimum | Menentukan kolom `NOT NULL` |
| D-8 | `resident_code` unik global atau per cabang | Menentukan bentuk UNIQUE index |
| D-9 | Dokumen tetap Google Drive atau pindah `internal/storage` | Menentukan isi `resident_documents` |
| D-10 | PII at-rest: plaintext / enkripsi kolom | Menentukan tipe & migrasi kolom |

> WO-01 & WO-02 dapat jalan dengan usulan SPEC. WO-03 ke atas **wajib** D-1, D-8, D-10 sudah diputuskan.

## Prinsip Pelaksanaan
1. **Semua kode fitur di `plugins/resident/`.** Tidak ada kode fitur di `internal/`.
2. **Skema lewat migrasi versioned** (`plugins/resident/migrations/postgres/`). **Nol `CREATE TABLE` di kode Go.**
3. **Permission `resident.*` di-seed lewat migrasi plugin `auth`** — tanpa itu semua route 403 (fail-closed).
4. **Registrasi plugin manual** di `cmd/server/main.go` dan `cmd/console`.
5. **Lingkungan build kanonik = container `app`** (`/usr/local/go` + `/go/bin/swag`).
6. Setiap WO wajib lulus `go build ./...`, `go vet ./...`, `go test ./...`, `gofmt -l`.
7. **Tidak ada push** ke remote.

## Daftar WO

| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `WO-01-plugin-skeleton.md` | Kerangka plugin `resident` + registrasi `server`/`console` + CLI/route stub + test kosong | — | draft |
| 2 | `WO-02-migrasi-skema-canonical.md` | Migrasi tabel canonical (9 tabel) + 5 tabel referensi + seed nilai baku | WO-01 | draft |
| 3 | `WO-03-permission-auth.md` | Migrasi plugin `auth`: seed permission `resident.read/write/delete` + pemetaan ke grup `root`/`admin`/`viewer` | WO-02 | draft |
| 4 | `WO-04-api-admin-crud.md` | Model GORM + service (validasi `branch_key`, enum, kode RM) + admin API CRUD ber-guard permission | WO-03, D-1/D-8/D-10 | draft |
| 5 | `WO-05-cli-resident.md` | CLI `resident list/show/add/update/status/documents` + validasi & output tabel | WO-04 | draft |
| 6 | `WO-06-importer-csv.md` | Importer CSV Notion → canonical sesuai §6 SPEC + laporan validasi + dry-run | WO-04 | draft |
| 7 | `WO-07-swagger-docs.md` | Anotasi swagger, regenerasi `docs/swagger`, dokumentasi `.env.example` (`INSTANCE_KEY`) & README | WO-06 | draft |

## Definisi Selesai (semua WO)
- `go build ./...` / `go vet ./...` / `go test ./...` lulus di container `app`.
- `gofmt -l` kosong.
- Migrasi `up` dan `down` keduanya terbukti jalan pada DB bersih.
- Tidak ada `CREATE TABLE` di kode Go.
- Tidak ada kode fitur di `internal/`.
- `bin/` di-rebuild **setelah** commit.
- Dokumen historic: `historic rebuild --json` → `historic save <id> --json` → `historic git commit`.

## Catatan Risiko

| # | Risiko | Mitigasi |
|---|---|---|
| R-1 | Permission belum di-seed → semua route 403 | WO-03 wajib sebelum WO-04 diuji; uji dengan grup `root` |
| R-2 | Data sumber tidak seragam (tanggal 2 format, `"2600 gram"`, `"-"`, RM+URL) | Normalisasi eksplisit di WO-06 sesuai §6.2 SPEC |
| R-3 | Nama fasilitas ditulis berbeda (`RS. Premagana` vs `Rumah Sakit Umum Premagana`) | `facility_aliases` + alias matching case/space-insensitive |
| R-4 | `resident_code` bukan generator otomatis; 19/50 kosong di sumber | Impor **menerima** kode kosong sebagai `NULL` sementara? → butuh keputusan D-8; generator kode masuk WO terpisah |
| R-5 | PII (NIK/BPJS) terlanjur tersimpan plaintext | Jangan implementasi kolom PII sebelum D-10 diputuskan, atau siapkan migrasi enkripsi |
| R-6 | FK `changed_by` ke `admins` lintas-plugin | Tanpa FK constraint (validasi di service), atau FK bila urutan migrasi dijamin |
