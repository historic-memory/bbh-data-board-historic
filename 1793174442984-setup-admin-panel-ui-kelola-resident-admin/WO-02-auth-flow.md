---
title: "WO-02: Auth Flow SPA"
description: "Perbaiki login (GAP-1), simpan main_token, muat profil + permissions, logout, penanganan 401 global, dan halaman Login yang layak."
status: draft
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
