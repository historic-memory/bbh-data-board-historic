---
title: "WO-06: UI Admin & RBAC"
description: "Halaman admin: daftar admin + assign/unassign grup, serta halaman grup & permission (read-only) dengan penyesuaian tombol berbasis permission."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - ui
  - rbac
---

# WO-06: UI Admin & RBAC

## Referensi
- SPEC: A-5, A-10, A-14, AC-5.
- Routes & guard:
  - `GET /api/admins` (`admin.read`), `GET /api/admins/:id` (`admin.read`)
  - `GET /api/admins/:id/groups` (`group.read`)
  - `PUT /api/admins/:id/groups` body `{group_keys: string[]}` (`group.write`)
  - `GET /api/groups` (`group.read`) → `{items: GroupView[]}`
  - `GET /api/permissions` (`permission.read`) → `{items: Permission[]}`
- `AdminView{id,name,email,is_active,groups[],created_at,updated_at}`;
  list admin **sudah** memakai `adminViewWithGroups` (groups terisi).
- `GroupView{id,key,name,description,is_system,is_default,is_active,permissions[]}`.
- Anti-escalation server: aktor tidak bisa memberi permission yang tidak dimilikinya.

## Langkah
1. `src/features/rbac/{types.ts,api.ts}` untuk admin/grup/permission.
2. Halaman `/admins`: tabel (nama, email, status aktif, badge grup), pencarian
   lokal, tombol "Kelola Grup" (`group.write`).
3. Modal Kelola Grup: checkbox daftar grup (dari `GET /api/groups`), prefill dari
   `GET /api/admins/:id/groups`, simpan `PUT` dengan **seluruh** `group_keys`.
   Tampilkan peringatan bila grup sistem `root` (`permissions: ["*"]`) dipilih
   dan aktor bukan pemegang `*`. Tangani 403/409 (anti-lockout) dengan pesan jelas.
4. Halaman `/groups`: kartu/tabel grup (badge `sistem`/`default`), daftar
   permission per grup (read-only). Tanpa create/update/delete (A-14).
5. Halaman/drawer `/permissions`: katalog permission read-only, dikelompokkan
   per prefix (`resident.*`, `admin.*`, `group.*`, `permission.*`, `*`).
6. Navigasi sidebar menyembunyikan menu bila permission terkait tidak dimiliki.

## Kriteria Penerimaan
- [AC-1] Daftar admin tampil dengan grup masing-masing (bukan kosong).
- [AC-2] Assign/unassign grup berhasil; perubahan terlihat setelah refetch.
- [AC-3] Anti-lockout: melepas grup `*` terakhir → 409 → pesan Indonesia yang jelas.
- [AC-4] Pengguna tanpa `group.write` tidak melihat tombol "Kelola Grup"; 403
  tetap ditangani jika dipaksa.
- [AC-5] Penolakan anti-escalation (403) ditampilkan sebagai pesan, bukan crash.
- [AC-6] Halaman grup & permission hanya-baca (tidak ada aksi mutasi).
- [AC-7] `npm run build` lulus.

## Bukti
Screenshot list admin, modal kelola grup, penolakan anti-lockout (409), dan viewer
tanpa `group.write`.

## Catatan
Create/update/delete admin & grup sengaja di luar lingkup (A-14); jangan
menambahkan tombol yang tidak berfungsi.
