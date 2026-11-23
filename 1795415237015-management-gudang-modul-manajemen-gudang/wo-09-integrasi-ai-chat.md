---
title: WO-09 — Integrasi AI chat (plugin ingest)
description: "Segmen wh: baca stok via chat, usul transaksi, terapkan"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-09 — Integrasi AI chat (plugin `ingest`)

## Prasyarat

WO-08 lulus; keputusan O-3 owner.

## Tujuan

EX-2 owner: modul gudang bisa dibaca/diusul lewat AI chat, memakai pola topic `1795107489593` (segment tools) — STRICTNESS tetap di Go (D-11 topic ingest).

## Lingkup (sesuai O-3 final)

- **Baca** — segmen `wh` di whitelist `segment_read` (`plugins/ingest/services/segment_read.go`): endpoint GET-only (summary/batches/items) fail-closed; stok bukan PII → tanpa deidentify; paginasi + `truncated`; guard `warehouse.read` pre-check.
- **Tulis** — whitelist TULIS terpisah di `segment_write_propose` (endpoint+method eksplisit, usulan hanya `POST /api/admin/wh/batches` dan `POST /api/admin/wh/movements`): kartu usulan di chat → user setujui (`confirmed_by_user` wajib) → `apply_record_actions` eksekusi HTTP self-call Bearer USER (RBAC+validasi asli jalan; stok kurang → 409 diteruskan ke chat).
- Master kategori/produk TIDAK di chat (dibuat via UI) — mencegah agent mengarang master.
- tool-schema Node: objek bebas WAJIB `loosenFreeFormFields` (gotcha `convertJsonSchemaToZodSchema`).
- agent.md: kontrak gudang (baca→usul→setujui→terapkan; PII tidak ada; gudang/produk harus dikenal).

## Acceptance Criteria

- AC-1 Chat: "stok minyak angin di gudang X?" → jawaban dari summary (bukan karangan).
- AC-2 Chat: usul masuk/keluar → kartu konfirmasi → setujui → data nyata; tolak usulan tanpa persetujuan.
- AC-3 Endpoint di luar whitelist → 400 fail-closed + audit `ingest_audit provider='ingest-tool'`.
- AC-4 Bruno `ingest-crud` diperluas segmen wh; regresi nol koleksi existing.

## Non-tujuan

Tool edit master, tool impor excel via chat (bisa jadi WO lanjutan bila owner mau).

## Catatan Implementasi (Dev)

(disi bertahap)

