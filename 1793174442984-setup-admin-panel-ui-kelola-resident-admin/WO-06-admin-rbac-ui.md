---
title: "WO-06: UI Admin & RBAC"
description: "Halaman admin: daftar admin + assign/unassign grup, serta halaman grup & permission (read-only) dengan penyesuaian tombol berbasis permission."
status: progress
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

## Hasil Implementasi (commit `49a8a92`)
- Berkas: `src/api/rbac.ts` (tipe + 5 fungsi), `pages/admin/admins/{AdminListPage,ManageGroupsModal}.tsx`,
  `pages/admin/rbac/{GroupsPage,PermissionsPage}.tsx`, rute di `App.tsx`, menu di `AdminLayout.tsx`.
- **Deviasi struktur (diterima):** rencana `src/features/rbac/` → aktual `src/api/rbac.ts` + `src/pages/admin/rbac/`,
  mengikuti konvensi WO-02..WO-04. Tidak perlu diubah.
- **409 dibedakan dari 409 generik:** `toApiFailure()` menerjemahkan 409 → "bentrok dengan data lain" (menyesatkan
  untuk anti-lockout). Modal memeriksa `statusOf(err) === 409` dan memberi pesan anti-lockout spesifik;
  403 memakai pesan anti-escalation.
- **Keterbatasan uji (dicatat jujur):** jalur **403 anti-escalation tidak terpicu lewat UI** karena aktor uji
  (root) memegang `*` → `EnsureCanGrant` meloloskan. Yang terbukti: viewer tanpa `group.write` tidak punya
  tombol, API PUT → 403, dan cabang pesan 403 ada serta tidak crash. Terbuka sebagai **H-8**.
- Read-only terverifikasi: `GroupsPage` & `PermissionsPage` tidak memiliki elemen `<button>`.
- Gate lain lulus: tabel admin grup terisi, modal prefill benar, assign viewer→admin tersimpan (refetch),
  anti-lockout 409 dengan modal tetap terbuka, build 130 modul, `grep` host:port di `dist` kosong.

## Laporan Implementasi (2026-10-28)

### File
| Berkas | Isi |
|---|---|
| `src/api/rbac.ts` (baru) | Tipe `AdminAccount`/`GroupView`/`PermissionEntry` + `listAdmins`, `listGroups`, `listPermissions`, `getAdminGroups`, `setAdminGroups` |
| `src/pages/admin/admins/AdminListPage.tsx` (baru) | Tabel admin (nama, email, status, badge grup) + pencarian lokal |
| `src/pages/admin/admins/ManageGroupsModal.tsx` (baru) | Modal kelola grup (checkbox, prefill, PUT seluruh daftar) |
| `src/pages/admin/rbac/GroupsPage.tsx` (baru) | Kartu grup + permission, hanya baca |
| `src/pages/admin/rbac/PermissionsPage.tsx` (baru) | Katalog permission per prefix, hanya baca |
| `src/App.tsx` (ubah) | `/admins`, `/groups`, `/permissions` menggantikan 2 placeholder |
| `src/layouts/AdminLayout.tsx` (ubah) | Menu “Permission” (`permission.read`) + judul dokumen |

Tidak ada perubahan di `internal/`, `plugins/`, atau build backend.

### Catatan implementasi
1. **Struktur berbeda dari rencana `src/features/rbac/`** — repo ini sudah punya
   konvensi `src/api/*` + `src/pages/admin/*` (WO-02..WO-04), jadi ditempatkan di
   sana agar konsisten. Deviasi dicatat, bukan diam-diam.
2. **Tabel admin memakai `adminViewWithGroups`** yang sudah ada di backend — grups
   terisi tanpa perlu request per-admin (tanpa N+1).
3. **Badge grup memakai nama**, bukan key mentah; grup ber-`*` diberi tone
   `warning` + label “akses penuh”. Bila `GET /api/groups` gagal (butuh
   `group.read`), tabel tetap tampil dengan key mentah (`.catch(() => [])`).
4. **Anti-lockout dibedakan dari 409 generik.** `toApiFailure()` menerjemahkan 409
   menjadi “bentrok dengan data lain” — pesan itu menyesatkan untuk kasus ini,
   jadi modal memeriksa `statusOf(err) === 409` dan menampilkan pesan khusus.
   Branch 403 memakai pesan anti-escalation (memberi izin yang tidak dimiliki /
   mengubah keanggotaan grup sendiri).
5. **Modal hanya dimount** bila `group.write` dimiliki (`{canWriteGroups && …}`),
   sehingga viewer tidak punya jalan ke UI mutasi (`dialog` = 0 di DOM).
6. **Peringatan wildcard di klien** muncul bila aktor mencentang grup ber-`*`
   tanpa memegang `*` — murni bantuan UX; keputusan tetap di server.
7. **Halaman grup & permission tidak punya satu pun tombol** (A-14) — diverifikasi
   dengan menghitung `main button` = `[]` di kedua halaman.

### Gate (browser, `http://100.125.175.34:5173`, data nyata)
Akun uji (`viewer-wo06@bbh.dev`) dibuat lalu **dihapus** setelah uji; keanggotaan
`root@bbh.dev` dan `ops@bbh.dev` **dikembalikan** ke kondisi semula.

| # | Uji | Hasil |
|---|---|---|
| 1 | `/admins` sebagai root | 4 baris; kolom Grup **terisi** (`Root`/`Viewer`), bukan kosong |
| 2 | Modal Ops | prefill benar: `viewer` tercentang, `admin`/`root` tidak |
| 3 | Assign viewer→admin | PUT body `{"group_keys":["admin"]}`; setelah refetch badge jadi **Administrator** |
| 4 | Anti-lockout (409) | `root@bbh.dev` dipindah ke `admin` (1 wildcard tersisa) → melepas `root` dari `root@admin.local` → **409**, modal tetap terbuka, pesan: “Gagal: tindakan ini akan menghilangkan satu-satunya admin dengan akses penuh (`*`)…” |
| 5 | Pemulihan | `root@bbh.dev` → `root`, `ops@bbh.dev` → `viewer`; daftar kembali seperti semula |
| 6 | Viewer tanpa `group.write` | tombol “Kelola grup” **tidak ada** (kolom Aksi `—`), `dialog` = 0 |
| 7 | Viewer dipaksa di API | `PUT /api/admins/:id/groups` → **403**; `GET /api/admins` & `GET /api/groups` → 200 |
| 8 | `/groups` | 3 kartu (Viewer/Administrator/Root), badge `sistem`/`default`/`akses penuh`; **0 tombol** |
| 9 | `/permissions` | 8 prefix (`*`, admin, audit, endpoint, group, member, permission, resident) = 17 permission; **0 tombol** |
| 10 | Menu sidebar | “Permission” muncul untuk root; menu tetap difilter `hasPermission` |
| 11 | `npm run build` | hijau (`tsc -b && vite build`, 130 modul) |
| 12 | Stateless | `grep -rEo "http://(100\.125\.175\.34\|localhost):(8082\|5173)" dist` → kosong |

### Bukti tambahan
- `PUT` mengirim **seluruh** `group_keys` (bukan diff) sesuai kontrak backend —
  terbukti dari body request yang direkam.
- 403 anti-escalation belum bisa diuji lewat UI oleh aktor non-`*` (root punya `*`,
  jadi `EnsureCanGrant` meloloskan). Yang terverifikasi: (a) viewer tanpa
  `group.write` memang tidak punya tombol, (b) API menolak 403, (c) branch pesan
  403 ada di kode dan tidak pernah muncul sebagai crash.

### Belum dikerjakan (memang di luar lingkup)
- Create/update/delete admin & grup (A-14). Akun tetap dibuat via CLI.
- Menonaktifkan admin dari UI (`is_active`) — backend punya `PATCH /api/admins/:id`,
  tapi panel sengaja belum memakainya.
- Menampilkan permission EFEKTIF per admin (butuh `EffectivePermissions` per baris;
  sengaja dihindari agar daftar tidak N+1).
