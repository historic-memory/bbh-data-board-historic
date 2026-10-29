---
title: 'SPEC: Navigasi Admin — pengelompokan menu per domain/plugin (outline group)'
description: 'Restrukturisasi sidebar admin menjadi outline group per domain bisnis (plugin), dengan registry navigasi terpusat yang data-driven sehingga domain/plugin baru cukup menambah entri — tanpa menyentuh layout.'
status: draft
created: "2026-10-29"
updated: "2026-10-29"
tags:
    - admin-panel
    - ui
    - navigation
    - frontend
---
# SPEC: Navigasi Admin — Pengelompokan Menu per Domain/Plugin

## Ringkasan
Sidebar admin saat ini **flat**: 11 item sejajar (`AdminLayout.tsx` → `NAV_ITEMS`), label
"Kebutuhan RT — …" diulang 5×, dan menu administrasi (RBAC) terbaur dengan menu data. Ketika
jumlah plugin/domain bertambah (mis. `elec`, kebutuhan lain, laporan), daftar akan makin panjang
dan sulit dipindai.

Tujuan: mengubah sidebar menjadi **outline group per domain bisnis (= plugin)**, dengan
**registry navigasi terpusat yang data-driven**. Menambah domain/plugin baru nanti = cukup
menambah satu entri di registry, **tanpa menyentuh struktur layout**.

> Arah owner (2026-10-29): *"lebih ke plugin core business-nya; nanti bisa expand, atau bikin
> outline group"*. Keputusan teknis bentuknya diserahkan ke Manager (dokumen ini).

## Kondisi Awal (terverifikasi)
- `sub_app/webapp/src/layouts/AdminLayout.tsx`:
  - `NAV_ITEMS: NavItem[]` — array **datar**; tiap item `{ to, label, permission? }`, difilter
    hanya oleh `hasPermission(permission)`.
  - `TITLES` — array datar terpisah untuk `document.title` (duplikasi sumber dengan `NAV_ITEMS`).
  - Sidebar merender `items.map(...)` sebagai `<NavLink>` sejajar; **tidak ada heading/section**.
- `src/App.tsx` — rute sudah dikelompokkan per area secara komentar, tetapi URL tetap **flat**
  (`/residents`, `/facilities`, `/admins`, `/groups`, `/permissions`, `/needs/*`).
- `src/auth/AuthContext.tsx` — `hasPermission(key)` (pemegang `*` = true) dan `user.permissions`.
- Halaman per domain sudah ada di `src/pages/admin/{residents,facilities,admins,rbac,needs}`.

## Ruang Lingkup
- Registry navigasi terpusat (single source of truth menu) yang data-driven & mudah di-expand.
- Rendering sidebar sebagai **outline group** (section heading + item), termasuk perilaku
  buka/tutup (collapsible) dan penyorotan section aktif.
- Filter izin per section **dan** per item; section kosong tidak dirender.
- `document.title` diturunkan dari registry yang sama (hapus duplikasi `TITLES`).
- Dukungan perilaku mobile (drawer) tetap berjalan.

## Non-Goals
- **Tidak** mengubah URL/rute (`App.tsx` tetap; di luar scope WO ini — lihat N-6).
- **Tidak** mengubah backend, permission, atau RBAC.
- **Tidak** mengubah styling tema/Tailwind v4, komponen UI dasar, atau halaman per domain.
- **Tidak** menambah ikon pihak ketiga (ikon opsional memakai glyph/emoji bila perlu).
- **Tidak** menyentuh `/portal` dan halaman publik.

## Keputusan Desain
| # | Keputusan | Nilai |
|---|---|---|
| N-1 | **Arti "group"** | Kelompok **domain/plugin bisnis** (fungsional), **bukan** RBAC group. |
| N-2 | **Sumber menu** | **Registry terpusat** `src/navigation/menu.ts` sebagai *single source of truth*; `AdminLayout` hanya merender. |
| N-3 | **Bentuk UI** | **Outline group**: setiap group punya heading label + blok item. Group **collapsible** (bisa buka/tutup). |
| N-4 | **Section & isi** | `Utama` (Dashboard) · `Data Warga` (Resident, Fasilitas) · `Kebutuhan` (sub-menu hsh) · `Administrasi` (Admin, Grup & Permission, Permission). |
| N-5 | **Filter izin** | Group tampil bila **minimal satu** item-nya lolos `hasPermission`; item tampil per izin item. Group tanpa item → tidak dirender. Item tanpa `permission` → selalu tampil. |
| N-6 | **URL/rute** | **Tidak diubah** di WO ini. Perapian URL per domain dicatat sebagai GAP-2 (kandidat follow-up), bukan syarat. |
| N-7 | **State buka/tutup** | Disimpan di `localStorage` (persist antar reload). Group yang memuat rute aktif **selalu terbuka**. |
| N-8 | **Nesting** | Model mendukung **2 level** (group → item). Sub-sub-level tidak diperlukan; "Kebutuhan RT" jadi satu group berisi 5 item. |
| N-9 | **Judul dokumen** | Diturunkan dari registry (`document.title` = label item + `BASE_TITLE`), menghapus array `TITLES` terpisah. |
| N-10 | **Kontrol akses tetap di server** | Registry hanya **menyembunyikan** menu; otorisasi nyata tetap ditegakkan backend (fail-closed). Menyembunyikan menu **bukan** kontrol keamanan. |
| N-11 | **Perluasan** | Domain/plugin baru = tambah satu objek group di registry; **tidak** menyentuh `AdminLayout`/`App.tsx`. Ini kriteria utama "bisa expand". |

## Rancangan Teknis (indikatif, bukan kode final)

Bentuk data registry (usulan):
```ts
export type NavItem  = { to: string; label: string; permission?: string }
export type NavGroup = { id: string; label: string; permission?: string; items: NavItem[] }
export const NAV_GROUPS: NavGroup[] = [ /* Utama, Data Warga, Kebutuhan, Administrasi */ ]
```
`AdminLayout` menghitung `visibleGroups` dari `NAV_GROUPS` + `hasPermission`, lalu merender
per group: header (tombol toggle) + daftar `<NavLink>`. Judul dokumen diambil dari item yang
cocok dengan `location.pathname` (helper di registry, mis. `titleFor(pathname)`).

## GAP
| # | GAP | Catatan |
|---|---|---|
| GAP-1 | `TITLES` duplikat dengan `NAV_ITEMS` | Diselesaikan oleh N-2/N-9 (satu sumber). |
| GAP-2 | URL masih flat, tidak mencerminkan domain | **Di luar scope** (N-6); follow-up bila owner minta. |
| GAP-3 | Belum ada uji otomatis komponen sidebar | Verifikasi di WO-02 (manual terarah + build). |
| GAP-4 | Label group berpotensi tidak konsisten dengan "plugin core business" | Label diputuskan saat review UI. |

## Pertanyaan Terbuka
- **O-1** Persistensi `localStorage` (N-7) diperlukan, atau cukup selalu terbuka?
- **O-2** Perlu ikon per group? (usulan: tidak, agar tidak menambah dependensi.)

## Rencana Work Order
| # | File | Scope | Status |
|---|---|---|---|
| WO-01 | `WO-01-registry-navigasi-outline.md` | Registry navigasi + render outline group di `AdminLayout` | pending |
| WO-02 | `WO-02-verifikasi-dokumentasi.md` | Verifikasi build/perilaku + dokumentasi | pending |
