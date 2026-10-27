---
title: "Work Order (Master Index): Import Data Penerimaan Resident"
description: "Rencana implementasi importer data penghuni BBH: baca CSV Notion (raw) atau template BBH (WO-08), staging + idempotensi, promosi ke skema canonical, dan laporan validasi. Dipisahkan dari topic `resident` (dulu WO-06) karena lingkupnya berdiri sendiri."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---
# Work Order (Master Index): Import Data Penerimaan Resident

## Referensi
- SPEC data core: `.historic/1793093835163-analisa-data-mentah-business/SPEC.md` (§6 pemetaan sumber)
- Plugin resident (canonical + API): `.historic/1793093835163-analisa-data-mentah-business/`
- Data mentah: `Data Bayi Yayasan 1b590f1a8a518093ac5bd1d366590b33_all.csv` (50 baris, 23 kolom) — di topic resident
- Topic AI-ETL (di luar lingkup): `.historic/1793096602507-integrasi-llm-ai-etl-copilotkit-jobmanager-sandbox-acp-agent/`

## Latar
Topic ini menerima **satu WO** yang sebelumnya bernama **WO-06** di topic
`1793093835163-analisa-data-mentah-business`. WO itu ditunda saat fokus ke
swagger/template/validator. Karena lingkupnya (importer end-to-end) cukup besar
dan mandiri, ia dipindah ke topic terpisah.

## Prinsip Pelaksanaan
1. Semua kode fitur di `plugins/resident/` (id plugin tetap `resident`).
2. Skema (termasuk staging table) lewat migrasi versioned — **nol `CREATE TABLE` di kode Go**.
3. Pakai ulang aturan validasi WO-09 & generator kode WO-10 (jangan gandakan).
4. Lingkungan build kanonik = container `app`.
5. Setiap WO wajib lulus `go build ./...`, `go vet ./...`, `go test ./...`, `gofmt -l`.
6. Tidak ada push ke remote.

## Daftar WO
| # | File | Scope | Prasyarat | Status |
|---|---|---|---|---|
| 1 | `wo-01-importer-csv.md` | Importer CSV (raw) + template (XLSX/CSV): staging, idempotensi (`resident_code`→`nik`), promosi canonical, laporan, dry-run — **via HTTP API** | WO-04/08/09 (topic resident, complete) | draft |

## Definisi Selesai
- `go build ./...` / `go vet ./...` / `go test ./...` lulus di container `app`; `gofmt -l` kosong.
- Migrasi staging `up`/`down` terbukti jalan pada DB bersih.
- `50 = valid + ditolak`; impor ulang idempoten.
- Tidak ada DDL di kode Go; tidak ada kode fitur di `internal/`.
- Dokumen historic: `historic rebuild` → `historic save` → `historic git commit`.

