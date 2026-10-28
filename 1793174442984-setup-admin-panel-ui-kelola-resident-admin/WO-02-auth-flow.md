---
title: "WO-02: Auth Flow SPA"
description: "Perbaiki login (GAP-1), simpan main_token, muat profil + permissions, logout, penanganan 401 global, dan halaman Login yang layak."
status: complete
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - ui
  - auth
---

# WO-02: Auth Flow SPA

## Referensi
- SPEC: A-3, A-10, GAP-1, AC-1, AC-2, AC-6.
- `sub_app/webapp/src/api/client.ts` (`setMainToken`, `getMainToken`,
  `clearMainSession`, interceptor 401).
- Backend: `POST /api/admin/login` → `{access_token, expires_in}`;
  `GET /api/admin/me` (WO-01 B-1); `POST /api/admin/logout` (stateless).

## Kondisi Awal (masalah)
| # | Masalah |
|---|---|
| M-1 | `Login.tsx` memanggil `/api/auth/login` (tidak ada) → selalu 404 |
| M-2 | Token tidak pernah disimpan; hanya `setMsg(JSON.stringify(res.data))` |
| M-3 | Tidak ada state user, tidak ada redirect setelah login |
| M-4 | Tidak ada penanganan 401 terpusat di level UI (hanya interceptor axios) |

## Langkah
1. **AuthContext** (`src/auth/AuthContext.tsx`):
   - state `{ user: MeResponse | null, status: 'loading' | 'authenticated' | 'anonymous' }`
   - `login(email, password)`: POST `/api/admin/login` → `setMainToken(token)` →
     `fetchMe()` → set status.
   - `logout()`: POST `/api/admin/logout` (abaikan error) → `clearMainSession()`
     → status `anonymous`.
   - `hasPermission(key)`: `permissions.includes('*') || permissions.includes(key)`.
   - bootstrap saat mount: jika ada token → `fetchMe()`; 401 → bersihkan sesi.
2. **Login page** (`pages/Login.tsx`): form email+password, tombol disabled saat
   submit, pesan error Indonesia ("Email atau kata sandi salah" untuk 401,
   "Tidak dapat menghubungi server" untuk network error), redirect ke halaman
   asal (`location.state.from`) atau `/dashboard` setelah sukses.
3. **401 global**: daftarkan handler di `client.ts` (atau custom event
   `auth:unauthorized`) yang dipanggil interceptor → `clearMainSession()` +
   `navigate('/login')`. Hindari redirect loop saat request login sendiri gagal.
4. **Logout** tersedia di shell (dipakai WO-03) via `AuthContext.logout()`.
5. Hapus jalur lama `/api/auth` bila terbukti tidak dipakai (cek `ROUTE_OWNERS`
   — entri `/api/auth` ikut dihapus/dibiarkan? **putuskan di implementasi**;
   rekomendasi: hapus agar tabel owner tidak menyesatkan).

## Kriteria Penerimaan
- [AC-1] Login dengan kredensial valid → `main_token` ada di localStorage →
  redirect ke `/dashboard` (setelah WO-03: `/app/dashboard`).
- [AC-2] Kredensial salah → 401 → pesan ramah, token tidak tersimpan.
- [AC-3] Reload halaman saat masih login → profil dimuat ulang dari `/api/admin/me`.
- [AC-4] Token kedaluwarsa/di-revoke → request berikutnya 401 → sesi dibersihkan
  dan user diarahkan ke login **sekali** (tanpa loop).
- [AC-5] `permissions` dari `/me` tersimpan di context dan bisa dibaca lewat
  `hasPermission` (dipakai WO-04/WO-06).
- [AC-6] `npm run build` lulus; tidak ada `console.log` sisa debug.

## Bukti
Login sukses/gagal/luring (screenshot atau log), isi localStorage, dan
`tsc`/build hijau.

## Catatan
Backend logout stateless — UI hanya membuang token.

Verifikasi dilakukan lewat `npm run dev` (`http://localhost:5173/app/`) memakai
proxy dari **WO-00**; tidak perlu `npm run build` setiap iterasi.

## Bukti Pelaksanaan (2026-10-28)

### File
- **Baru** `sub_app/webapp/src/auth/AuthContext.tsx` — provider sesi tunggal:
  `status` (`loading|authenticated|anonymous`), `user`, `permissions`,
  `hasPermission`, `login`, `logout`, `refresh`.
- `sub_app/webapp/src/main.tsx` — `<AuthProvider>` **di dalam** `BrowserRouter`
  (handler 401 butuh `navigate`).
- `sub_app/webapp/src/api/client.ts` — `setUnauthorizedHandler()` +
  `handleLocalUnauthorized()`; interceptor respons dipasang pada `apiClient`
  **dan** `apiLocal`; entri mati `/api/auth` dihapus dari `ROUTE_OWNERS`
  (diganti entri `/api/groups` + `/api/permissions`).
- `sub_app/webapp/src/pages/Login.tsx` — ditulis ulang: form + state submit,
  pesan Indonesia, redirect ke `state.from` / `/dashboard`.
- `sub_app/webapp/src/pages/Dashboard.tsx` — request usang `GET /api/` (404)
  dihapus; menampilkan identitas sesi + tombol Keluar (isi ringkasan → WO-07).

### Anti-loop 401
`handleLocalUnauthorized` mengabaikan request ke `/api/admin/login` dan
`/api/admin/logout` (`isPublicAuthCall`). Jadi 401 dari login = "kredensial
salah", bukan "sesi berakhir" → tidak pernah memicu redirect. Handler global juga
melewati redirect bila sudah berada di `/login`.

### Gate lulus (browser, dev `http://localhost:5173/app/`)
| Skenario | Hasil |
|---|---|
| Buka `/app/login` | form tampil, title `BBH Data Board — Admin` |
| Login salah (`admin123` → kata sandi salah) | pesan **"Email atau kata sandi salah"**; `localStorage` = `{}` (token TIDAK disimpan); URL tetap `/app/login` (tanpa loop) |
| Login benar | redirect `/app/dashboard`; `localStorage` = `["main_token"]`; tampil `Masuk sebagai root (root@admin.local)`, `Grup: root`, `Izin: semua akses (*)` |
| Reload halaman | tetap di `/app/dashboard`, sesi hidup (`main_token` bertahan, `/me` dimuat ulang) |
| `main_token` diisi token palsu lalu buka `/app/dashboard` | 401 → sesi dibersihkan (`localStorage` = `[]`) → pindah ke `/app/login` **sekali**, `history.state.usr.from = "/dashboard"`, URL stabil setelah 1,2 s (tidak loop) |
| Klik **Keluar** | `main_token` hilang (`localStorage` = `[]`), kembali ke `/app/login` |
| `npm run build` | hijau (99 modul, `tsc -b && vite build`) |
| `grep -rEo "http://localhost:(8082|5173)" dist` | kosong |

### Catatan implementasi
- **Redirect ganda di dev**: React `StrictMode` menjalankan effect bootstrap dua
  kali sehingga log browser memperlihatkan **dua** 401 saat token palsu; redirect
  tetap terjadi **sekali** (terbukti URL stabil). Ini artefak dev-only.
- Handler 401 hanya membersihkan **sesi pusat**; `portal_token` (jalur cabang)
  tidak disentuh — sesuai pemisahan D-14/D-15.
- `plugins/federation/routes.go` (backend) masih memuat prefix `/api/auth` dan
  `federation_test.go` mengujinya sebagai `OwnerLocal`. **Tidak diubah** (R-1:
  `plugins/` hanya untuk B-1..B-3 di WO-01); entri itu hanya warisan yang tetap
  aman karena default owner = local. Dicatat sebagai temuan, bukan tindakan.
- Komponen individual **tidak** memanggil `hasPermission` sendiri-sendiri lagi;
  sumber tunggal adalah `useAuth()` (arahan Manager) — dikonsumsi WO-03/WO-04/WO-06.
