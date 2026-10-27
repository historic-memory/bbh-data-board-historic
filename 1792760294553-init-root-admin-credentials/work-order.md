---
title: work order
description: Buat admin root di DB dan simpan kredensial ke .credentials/admin.yaml
status: complete
created: "2026-10-23"
updated: "2026-10-23"
---
# Work Order: Buat admin root persisten + credentials file

## Goal
Buat admin root di database dan simpan kredensialnya ke `.credentials/admin.yaml`.

## Hasil
- Command yang dipakai: `console auth init-admin-root` (sudah tersedia dari plugin auth).
- Admin root dibuat di DB: email `root@admin.local`, password `admin123` (policy password min 8 karakter), `is_root=true`, `is_active=true`.
- Kredensial tersimpan di `.credentials/admin.yaml` (plaintext, permission 0600, gitignored).
- Tidak ada perubahan kode di scope WO ini.

## Verifikasi
- `cat .credentials/admin.yaml` → email + password tertulis benar.
- Permission file: `-rw-------`.

