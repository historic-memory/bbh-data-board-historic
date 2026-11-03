---
title: Master Index Work Order — Modul Kunjungan (plugin visit)
description: Induk seluruh work order plugin visit. Urutan, status, dan bukti commit tiap WO.
type: work-order
status: progress
created: "2026-11-03"
updated: "2026-11-03"
---

# Master Index — Work Order Plugin `visit`

Sumber desain: `spec.md` (D-1..D-13 terkunci, O-1..O-7 terjawab). Analisa data: `notes.md`.

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01 | Skeleton plugin + migrasi 3 tabel + enum | complete | `d0ca37a` | `06d6a05` |
| WO-02 | Permission + CRUD visitors & visits + dedup phone | complete | `2019f78` | `85cea34` |
| WO-03 | Riwayat status kunjungan (append-only) | complete | `fd932bd` | `9637280` |
| WO-04 | Generator template impor (xlsx/csv) | complete | `f6af156` | `702459f` |
| WO-05 | Validator umpan balik | complete | `adb20bb` | `84d0d37` |
| WO-06 | Importer 2 fasa + idempotensi | pending | — | — |
| WO-07 | UI admin kunjungan | pending | — | — |
| WO-08 | Swagger + README + Bruno + RBAC e2e | pending | — | — |
| WO-09 | Impor data awal 1.622 baris + laporan kualitas | pending | — | — |

**Urutan:** WO-01 → WO-02 → WO-03 → WO-04 → WO-05 → WO-06 → WO-07 → WO-08 → WO-09.

**Aturan lintas WO (pinned):**

- Kode fitur hanya di `plugins/visit/`; migrasi permission di plugin `auth` (fail-closed).
- Enum statis di `schema/fields.yaml` + konstanta Go + CHECK constraint; tanpa `ref_*`.
- Semua list API pakai kontrak `{items: [], total}`; UUIDv7 TEXT; audit `created_at/updated_at`.
- Hati-hati pola GORM `default:` (lihat catatan B-1b topic hsh): kolom bool/ber-default wajib test create zero-value.
- Token Bruno cepat kedaluwarsa setelah restart server; selalu ambil token baru sebelum `bru run`.
