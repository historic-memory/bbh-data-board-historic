---
title: work order
description: "Master index WO modul Management Gudang (plugin wh)"
status: progress
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
    - gudang
---

# Master Index — Work Order Modul Management Gudang

Plugin `plugins/warehouse` (ID `wh` di tabel/endpoint; permission `warehouse.*`). SPEC terkunci di `spec.md` (D-1..D-12); sumber kebutuhan = `conversation.md` + `notes.md` (M-1..M-5, EX-1, EX-2).

## Prinsip pinned

- Plugin di `plugins/warehouse/` — TIDAK menyentuh `internal/` (aturan user).
- Tanpa sinkronisasi antar gudang (M-3): keluar/masuk dianggap event lokal per instance.
- Stok = agregasi batch (`SUM(qty_remaining)`), tanpa kolom counter.
- Enum statis di kode + CHECK; template dari `schema/fields.yaml`; satuan dropdown REFERENSI.
- Jalur Excel lengkap (D-9): generator → validator → importer staging.
- Integrasi AI chat (D-10) = segmen `wh` di plugin `ingest`, pola topic `1795107489593`; strictness tetap di Go.
- **Log implementasi WAJIB** per WO (section `## Catatan Implementasi (Dev)`, ±10-20 entri `berkas:baris` + kenapa).

## Daftar Work Order

| WO | File | Status |
|---|---|---|
| WO-01 | `wo-01-skeleton-migrasi.md` | pending |
| WO-02 | `wo-02-permission-master.md` | pending |
| WO-03 | `wo-03-transaksi-stok.md` | pending |
| WO-04 | `wo-04-generator-template.md` | pending |
| WO-05 | `wo-05-validator.md` | pending |
| WO-06 | `wo-06-importer.md` | pending |
| WO-07 | `wo-07-ui-admin.md` | pending |
| WO-08 | `wo-08-swagger-bruno.md` | pending |
| WO-09 | `wo-09-integrasi-ai-chat.md` | pending |

Urutan: WO-01 → WO-02 → WO-03 → (WO-04 → WO-05 → WO-06) → WO-07 → WO-08 → WO-09.

## Micro-decision terbuka

O-1/O-2 (WO-06 impor: auto-create kategori/produk; gudang fail-closed), O-3 (WO-09 lingkup tool). Default usulan ada di `spec.md` §8; konfirmasi owner saat eksekusi WO terkait.

## Bukti & commit

(diisi per WO saat selesai)

