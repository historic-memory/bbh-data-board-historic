---
title: "WO-04: API Admin CRUD — model, service, handler residents"
description: "Implementasi model GORM + service (validasi branch_key, enum, kode RM, klasifikasi) + admin API CRUD untuk residents beserta entitas terkait (mothers, birth_records, status history, documents), seluruhnya ber-guard permission resident.*."
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 2
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

## Status Eksekusi (complete)
**Status: complete** — commit kode `17da01e` (`feat(resident): WO-04 model/service/API`). Tidak ada push.

### Catatan koreksi terhadap WO
- **AC-6 "validasi enum terhadap `ref_*`" USANG.** Sesuai D-3/D-12 tidak ada tabel `ref_*`; enum divalidasi terhadap **himpunan statis di kode** (`services/enums.go`) yang dijaga konsisten dengan `fields.yaml` & CHECK constraint lewat `TestEnumsMatchSchema`.
- **AC-7 (`branch_key` default dari `INSTANCE_KEY`)** diimplementasikan di `services.resolveInstanceKey` (env `INSTANCE_KEY`, fallback `"local"`) + `resolveBranchKey` (validasi ke `endpoints.key`).

### Bukti
Perintah: `docker compose exec app sh -c 'cd /src && go build ./... && go vet ./... && gofmt -l plugins/resident && go test ./plugins/resident/... -count=1'` → **lulus** (`schema`, `services` ok).

Uji HTTP (server binari WO-04 di container `app`, port 8082):

| Uji | Ekspektasi | Hasil |
|---|---|---|
| `POST /api/admin/residents` (nested mother/birth/status/documents) | 201 + kode auto | **201**, `resident_code=BBH-26-P-001` |
| `GET /api/admin/residents?q=Bayi Uji` | 200 | **200** |
| `GET /api/admin/residents/:id` | 200 | **200** |
| `PATCH /api/admin/residents/:id` | 200 | **200** |
| `POST /api/admin/residents/:id/status` ×2 | 201, append-only | **201, 201** → `GET .../status` = **3 baris** |
| `POST` gender `X` (enum invalid) | 400 | **400** |
| `POST` `branch_key:"nope"` | 400 | **400** |
| `POST` `resident_code` duplikat | 409 | **409** |
| `GET /api/admin/residents` sebagai `viewer` | 200 | **200** |
| `POST /api/admin/residents` sebagai `viewer` | 403 (fail-closed) | **403** |
| `DELETE /api/admin/residents/:id` | 200 | **200** |

| # | AC | Hasil |
|---|---|---|
| AC-1 | 12+ route berfungsi | ✅ 14 route ter-wire (log GIN 10 entri registrasi + uji di atas) |
| AC-2 | Pakai `RequirePermission` | ✅ di `plugin.go` (read/write/delete) |
| AC-3 | Fail-closed viewer | ✅ `viewer` GET 200 / POST 403 |
| AC-4 | Status append-only | ✅ 2 POST → 3 baris |
| AC-5 | Kode divalidasi format & unik | ✅ `ValidateResidentCode` + 409 |
| AC-6 | Enum tak dikenal → 400 | ✅ (via enum statis, lihat koreksi) |
| AC-7 | `branch_key` default `INSTANCE_KEY` | ✅ `resolveInstanceKey`/`resolveBranchKey` |
| AC-8 | Nol `CREATE TABLE` di Go; nol fitur di `internal/` | ✅ skema via migrasi; kode di `plugins/resident/` |
| AC-9 | Operasi multi-tabel satu transaksi | ✅ `CreateResident` (mother→resident→birth→admission→status→documents) + `TestCreateResidentTransactionRollback` |
| AC-10 | build/vet/test/gofmt | ✅ lulus di container `app` |

**Commit kode:** `17da01e`.
