---
title: "WO-05: CLI resident — list/show/add/update/status/documents"
description: "Perintah console untuk mengelola data core residents dari terminal: list, show, add, update, status (tambah riwayat), documents, serta perintah referensi (facility, ref). Konsisten dengan pola CLI plugin endpoint & auth."
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
---

# WO-05: CLI `resident`

**SPEC acuan:** `SPEC.md` §4, §5, §6
**Prasyarat:** WO-04

## Cakupan (`plugins/resident/cli.go`)

```
resident list        [--branch KEY] [--status CODE] [--q NAMA] [--page N] [--limit N]
resident show        <id|resident_code>
resident add         --name NAMA --gender L|P --birth-date YYYY-MM-DD [--mother-nama --mother-nik]
                     [--weight N --length N --child-order N --delivery NORMAL --attendant DOKTER]
                     [--facility NAMA] [--branch KEY] [--admitted-at YYYY-MM-DD] [--status CODE]
resident update      <id> [--name ...] [--nik ...] [--bpjs ...] ...
resident status      <id> --code CODE [--effective YYYY-MM-DD] [--partner JCH] [--note ...]
resident documents   <id> [--add --type drive_folder --url URL --label ...]
resident mother      list | show <id>
resident facility    list | add --name NAMA --type hospital [--alias ALIAS]
```

## Aturan
- Output tabel rapi bila TTY; `--json` untuk mode mesin (pola `historic`-like envelope tidak wajib, cukup JSON daftar).
- **Tidak** melakukan DDL; hanya memanggil service WO-04.
- Perintah destruktif (bila ada) wajib konfirmasi atau flag eksplisit.
- Perintah terdaftar via `ConsoleCommands()` (sudah disiapkan di WO-01).

## Bukti yang Harus Dilaporkan
- Transkrip: `resident add` satu bayi → `resident list` menampilkannya → `resident show <code>` detail lengkap.
- `resident status` dua kali → `resident show` menampilkan status terkini + riwayat.
- `resident facility add` dengan alias `RS. Premagana` → `resident show` menampilkan fasilitas baku.
- `resident list --json` valid (bisa `jq`).
- `go build`/`vet`/`test`/`gofmt` lulus.
- Commit hash.

## Kriteria Penerimaan
| # | AC |
|---|---|
| AC-1 | Semua subcommand di atas ada & berfungsi. |
| AC-2 | `resident show` menerima `id` (UUID) **atau** `resident_code`. |
| AC-3 | `resident add` membuat resident + mother + birth_record + status awal dalam satu transaksi. |
| AC-4 | `resident status` **menambah** riwayat, tidak menimpa. |
| AC-5 | `--json` menghasilkan JSON valid. |
| AC-6 | Tidak ada DDL & tidak ada logika domain baru (memakai service). |
| AC-7 | `go build`/`vet`/`test`/`gofmt` lulus. |
