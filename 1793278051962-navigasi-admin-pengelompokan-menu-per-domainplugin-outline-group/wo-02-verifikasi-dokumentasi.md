---
title: "WO-02: Verifikasi & dokumentasi navigasi outline group"
description: "Memverifikasi perilaku sidebar hasil WO-01 (filter izin, collapsible, persistensi, mobile) dan mendokumentasikan cara menambah domain/plugin baru di registry."
status: complete
created: "2026-10-29"
updated: "2026-10-29"
tags:
  - work-order
  - admin-panel
  - ui
  - navigation
  - verification
---

# WO-02: Verifikasi & Dokumentasi

## Referensi
- SPEC `spec.md` (khususnya N-5, N-7, N-10, N-11).
- Hasil WO-01 (`AdminLayout.tsx` + `src/navigation/menu.ts`).

## Tujuan
Membuktikan perilaku sidebar hasil WO-01 benar, dan mendokumentasikan cara menambah domain/plugin
baru sehingga sifat "bisa expand" dapat dipakai orang lain tanpa membaca kode layout.

## Deliverable
1. **Verifikasi perilaku** (checklist, dengan bukti singkat):
   - Filter izin per group & per item (uji dengan user viewer tanpa sebagian izin).
   - Group kosong tidak dirender.
   - Collapsible: buka/tutup + persist setelah reload; group rute aktif terbuka.
   - `document.title` benar di beberapa rute.
   - Drawer mobile berfungsi.
2. **Dokumentasi singkat** (di `sub_app/webapp/README.md` atau komentar registry):
   - Cara menambah domain/plugin baru: contoh langkah (daftarkan rute di `App.tsx` + tambahkan
     `NavGroup` di `menu.ts`).
   - Peringatan N-10: menyembunyikan menu bukan kontrol keamanan.

## Kriteria Penerimaan (AC)
- **AC-1** Checklist verifikasi terisi dengan hasil nyata (bukan asumsi), termasuk ≥1 skenario izin terbatas.
- **AC-2** Terdapat dokumentasi "cara menambah domain" yang bisa diikuti tanpa membuka `AdminLayout.tsx`.
- **AC-3** `npm run build` tetap sukses setelah perubahan dokumentasi (bila menyentuh kode).
- **AC-4** Tidak ada regresi: rute lama tetap dapat diakses lewat sidebar.

## Bukti yang Diminta
- Checklist hasil verifikasi (dengan perintah/nilai aktual).
- Tautan/ringkasan dokumentasi yang ditambahkan.

## Hasil Verifikasi (2026-10-29)

Lingkungan: dev server Vite `http://localhost:5173/app/` (HMR), backend `:8082`,
diuji via Playwright (tab remote). Kode commit `c458f82`.

| # | Skenario | Hasil aktual | Status |
|---|---|---|---|
| 1 | Render group (akun root) | Sidebar = 4 group: **Utama** (Dashboard), **Data Warga** (Resident, Fasilitas), **Kebutuhan** (Entri/Periode/Item/Kategori/Template), **Administrasi** (Admin, Grup & Permission, Permission). Label "Kebutuhan RT — …" tidak lagi diulang. | ✅ AC-1 |
| 2 | Tambah domain tanpa sentuh layout | Tambah sementara 1 `NavGroup` (`Probe AC2`) → grup + item muncul; hapus → kembali normal. `grep` layout tanpa item hard-code. | ✅ AC-2 |
| 3 | Filter izin (group kosong) | Akun `navlimited@test.local` ber-izin **hanya `hsh.read`** (via `auth group create/assign`): sidebar = **Utama** + **Kebutuhan** saja; **Data Warga** & **Administrasi** TIDAK dirender. Dashboard balas **403** (fail-closed backend). | ✅ AC-3 |
| 4 | Collapsible + persist + auto-open | Tutup "Kebutuhan": `aria-expanded=false`, item hilang, `localStorage["admin.nav.collapsed"]=["kebutuhan"]`. Reload → tetap tertutup. Kunjungi `/app/needs/entries` → group terbuka (`aria-expanded=true`) tanpa mengubah localStorage. | ✅ AC-4 |
| 5 | `document.title` | `/app/dashboard` → "Dashboard · …"; `/app/needs/entries` → "Entri · BBH Data Board — Admin"; tidak ada array `TITLES` duplikat. | ✅ AC-5 |
| 6 | Drawer mobile | Viewport 480px: tutup = aside `-translate-x-full` tanpa backdrop; buka = `translate-x-0` + backdrop hadir; `aria-expanded` toggle; tautan `NavLink` tetap berfungsi. | ✅ AC-6 |
| 7 | Build | `npm run build` SUKSES (147 modul). | ✅ AC-7 |

**Catatan label (GAP-4):** group diberi label **"Kebutuhan"** (sesuai SPEC N-4).
Makna "domain hsh muncul sekali" dipenuhi lewat judul group tunggal; label final
dapat disesuaikan saat review UI tanpa menyentuh layout.

**Dokumentasi ditambahkan:** `sub_app/webapp/README.md` → bagian
"Navigasi admin — registry terpusat (`src/navigation/menu.ts`)": struktur
`NavGroup`/`NavItem`, langkah menambah domain (rute di `App.tsx` + entri
`NAV_GROUPS`), perilaku terjamin, dan peringatan **menyembunyikan menu bukan
kontrol keamanan (N-10)**.

**Regresi:** rute lama (`/residents`, `/facilities`, `/admins`, `/groups`,
`/permissions`, `/needs/*`) tetap dapat diakses dari sidebar (AC-4 WO-02).
