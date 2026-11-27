---
title: konteks
description: Analisa alur portal saat ini + usulan integrasi login pusat & pemilih cabang ke dashboard admin (frontend-only)
status: complete
created: "2026-11-27"
updated: "2026-11-27"
---
# konteks

## Latar

- Topik induk: `1793013271539` (Federated Branch Endpoints — multi-instance, SSO-ready).
- Instance yang hidup (2026-11-27): bali `:8082` (domain `bbh.rolldev.my.id` via
  nginx), jogja `:8101`, semarang `:8102`, lombok `:8103`, java-children `:8104`
  — semua `/healthz` 200.
- Data cabang di pusat **sudah lengkap**: tabel `endpoints` berisi `local` +
  4 cabang dengan `base_url` Tailscale (`100.125.175.34:8101–8104`).
- Kolom `is_active` pada `endpoints` = penanda endpoint yang sedang dipakai
  (`POST /api/admin/endpoints/{key}/use`), bukan flag aktif/nonaktif; semua
  baris `status=active` dan langsung bisa dipilih.

## Kondisi UI sekarang (halaman `/app/portal`)

- Halaman utility polos, **bukan bagian shell dashboard admin**: ada snapshot
  localStorage (`main_token` / `portal_token` / `active_endpoint`), panel Log,
  alur 3 langkah: login pusat → masuk portal cabang → keluar portal.
- Daftar endpoint dimuat **setelah login** (`useEffect` menunggu
  `main_token`) → dropdown tampak kosong bila belum login; default terpilih =
  item pertama (`local`), cabang lain ada di dalam dropdown.
- State saat analisa (2026-11-27): `main_token=ADA` (root@admin.local),
  `portal_token` & `active_endpoint` KOSONG → belum pernah masuk portal.
- Kosmetik: label & tombol menempel tanpa spasi (`active_endpointKOSONG`,
  `Masuk PortalPanggil WhoAmI`).

## Fondasi backend (sudah siap, tidak perlu perubahan backend)

- `GET /api/admin/endpoints` → daftar cabang (sudah dikonsumsi
  `Portal.tsx:50`).
- `POST /api/admin/portal-token {audience}` (`Portal.tsx:85`; butuh izin
  `endpoint.write`).
- `client.ts` otomatis melampirkan `X-Portal-Token` pada jalur cabang
  (`:160`, `:196`).
- Route business `/api/business/whoami`; alur enrollment: kunjungan pertama
  403 `pending` → approve admin cabang (`GET /api/admin/sso-enrollments` →
  `POST .../:id/approve`) → 200.
- Akun mirror `origin=sso` **tanpa password** → tidak bisa login via form
  cabang; identitas dicocokkan per iss+sub (D-16); auto-provision `pending`,
  **never auto-approve** (D-18 B); grup default = minimal baca (D-25).

## Gap UX yang mau ditutup

1. Alur portal terpisah dari dashboard admin — user harus keluar ke
   `/app/portal`.
2. Tidak ada switcher cabang di shell admin (pilih cabang → portal-token →
   badge instance aktif).
3. Status "sedang di cabang apa" tidak terlihat (`active_endpoint` hanya di
   localStorage, tidak ada indikator di UI).
4. Halaman Portal polos + label kosmetik menempel.

## Usulan arah (frontend-only — menunggu WO)

- Switcher cabang di shell/header admin: pilih cabang → otomatis minta
  `portal-token` → badge "instance aktif" (mis. "Jogja Baby Home") di header.
- Login pusat tetap dari halaman login admin biasa; alur portal menyatu
  dengan dashboard.
- Halaman `/portal` dipertahankan sebagai **debug harness** (log + snapshot
  localStorage); sekalian bereskan label kosmetiknya.
- Handle 403 `pending` di UI: tampilkan status "menunggu persetujuan admin
  cabang" (bukan error mentah).

## Referensi

- Analisa screenshot sesi 2026-11-27 (`bbh.rolldev.my.id/app/portal`).
- Topik induk: `1793013271539`; topik instance cabang: `1795514642792`.
- README `~/workspaces/bbh-branch` (bootstrap, admin cabang, SSO flow).
