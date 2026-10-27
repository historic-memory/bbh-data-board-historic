---
title: "WO-04: API Admin CRUD — model, service, handler residents"
description: "Implementasi model GORM + service (validasi branch_key, enum, kode RM, klasifikasi) + admin API CRUD untuk residents beserta entitas terkait (mothers, birth_records, status history, documents), seluruhnya ber-guard permission resident.*."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-04: API Admin CRUD

**SPEC acuan:** `SPEC.md` §4, §5, §7 (view), §8 (constraint), §9 (PII)
**Prasyarat:** WO-03; **D-1, D-8, D-10 wajib sudah diputuskan**
**Beralih dari:** stub route WO-01 (501 → implementasi nyata)

## Cakupan

### 1. Model (`plugins/resident/model/models.go`)
GORM model untuk setiap tabel WO-02. UUIDv7 di-generate di service (pakai `internal/uuid`).

### 2. View (`plugins/resident/model/view.go`)
Query untuk `v_resident_current_status` (status terkini = `effective_at` terbesar) dan `v_resident_full` (join residents + birth_records + mothers + status + branch_key).

### 3. Service (`plugins/resident/services/service.go`)
- Validasi `branch_key` (harus ada di `endpoints.key`; set default dari env `INSTANCE_KEY` bila kosong).
- Validasi enum terhadap tabel `ref_*` (tolak nilai tak dikenal dengan pesan jelas).
- Validasi & **generator** `resident_code` (format `BBH-<YY>-<P|L>-<NNN>`); lihat D-8 untuk cakupan unik.
- Validasi NIK 16 digit & BPJS (bila D-10 sudah memutuskan kebijakan PII).
- Transaksi: membuat `resident` + `mother` + `birth_record` + status awal + dokumen dalam **satu transaksi**.
- Menambah baris `resident_status_history` saat status berubah (tidak menimpa).
- Klasifikasi `Keterangan` → `facility` + `attendant_detail` (aturan §6.3 SPEC) — minimal: util `ClassifyFacility`.

### 4. Handler (`plugins/resident/handlers/handlers.go`)
| Method | Path | Guard |
|---|---|---|
| GET | `/api/admin/residents` | `resident.read` |
| GET | `/api/admin/residents/:id` | `resident.read` |
| POST | `/api/admin/residents` | `resident.write` |
| PUT | `/api/admin/residents/:id` | `resident.write` |
| PATCH | `/api/admin/residents/:id` | `resident.write` |
| DELETE | `/api/admin/residents/:id` | `resident.delete` |
| POST | `/api/admin/residents/:id/status` | `resident.write` |
| GET | `/api/admin/residents/:id/status` | `resident.read` |
| GET/POST | `/api/admin/residents/:id/documents` | `resident.read` / `resident.write` |
| GET | `/api/admin/mothers` | `resident.read` |
| GET | `/api/admin/facilities` | `resident.read` |
| POST/PUT | `/api/admin/facilities` | `resident.write` |

Respons mengikuti pola plugin existing (`MessageResponse`/`ErrorResponse`).

## Non-Cakupan
- CLI (WO-05), importer CSV (WO-06), swagger (WO-07).
- Sinkronisasi lintas cabang.
- Endpoint khusus member (di luar lingkup).

## Bukti yang Harus Dilaporkan
- `go build`/`vet`/`test`/`gofmt` lulus.
- Uji curl (admin grup `root`): **create → list → get → patch → status → delete**; kutip request & respons JSON ringkas.
- Uji fail-closed: admin grup `viewer` → `GET` 200, `POST` **403**.
- Bukti **status history tidak ditimpa**: dua kali `POST /status` menghasilkan 2 baris.
- Bukti `resident_code` invalid/duplikat ditolak dengan pesan jelas.
- Bukti `branch_key` tak dikenal ditolak.
- Bukti transaksi: gagal di tengah → tidak ada baris parsial.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | 12+ route berfungsi sesuai tabel di atas dengan guard permission benar. |
| AC-2 | `RequirePermission` dipakai (bukan cek manual), sesuai pola plugin `endpoint`. |
| AC-3 | Fail-closed terbukti: `viewer` tidak bisa menulis. |
| AC-4 | `resident_status_history` bersifat append-only (riwayat terjaga). |
| AC-5 | `resident_code` divalidasi format & keunikannya. |
| AC-6 | Enum divalidasi terhadap `ref_*`; nilai tak dikenal → 400 dengan pesan jelas. |
| AC-7 | `branch_key` default dari `INSTANCE_KEY` bila tidak diberikan. |
| AC-8 | Tidak ada `CREATE TABLE` di kode Go; tidak ada kode fitur di `internal/`. |
| AC-9 | Semua operasi multi-tabel berjalan dalam satu transaksi. |
| AC-10 | `go build`/`vet`/`test`/`gofmt` lulus. |
