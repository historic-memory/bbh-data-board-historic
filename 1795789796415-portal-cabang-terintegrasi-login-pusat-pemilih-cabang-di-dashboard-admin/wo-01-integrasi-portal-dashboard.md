---
title: WO-01 — Integrasi portal ke dashboard admin (frontend-only)
description: Switcher cabang + badge instance aktif di header AdminLayout, memakai helper client.ts yang sudah ada; /portal tetap debug harness
status: review
created: "2026-11-27"
updated: "2026-11-27"
---
# WO-01 — Integrasi portal ke dashboard admin (frontend-only)

## Referensi

- `konteks.md` (topik ini) — analisa kondisi & gap UX.
- Topik induk `1793013271539`; desain D-13/D-14 (dua jalur token), D-16/D-18/D-25
  (enrollment SSO) — **sudah terimplementasi** di `client.ts` & plugin
  `portalsso`; WO ini murni menyusun UI-nya.
- Implementasi acuan alur: `sub_app/webapp/src/pages/Portal.tsx` (login →
  portal-token → whoami → exit). Jangan ubah perilakunya selain kosmetik.

## Prinsip batas (HARD CONSTRAINT — baca sebelum menulis kode)

1. **Frontend-only.** Tidak ada perubahan Go/backend, migrasi, atau
   `api-tests/`.
2. **Tidak menambah dependensi npm.** Dropdown/badge memakai pola komponen
   yang sudah ada (`src/components/ui`, pola `ToastProvider`).
3. **Tidak mengubah `src/api/client.ts`.** Semua helper sudah lengkap —
   dipakai apa adanya.
4. Tabel `ROUTE_OWNERS` di `client.ts` tidak berubah; halaman admin yang ada
   tidak disentuh (mereka otomatis ikut via `apiClient`).
5. **Jangan tulis `localStorage` langsung** — wajib lewat helper
   `setPortalSession` / `clearPortalSession` / `getActiveEndpoint` /
   `getMainToken` (sumber: `client.ts:14-108`).
6. Satu sesi portal pada satu waktu (single `active_endpoint` — desain D-14).
7. `/app/portal` tetap berperilaku identik; hanya perbaikan spacing kosmetik.

## File

| Aksi | Path | Catatan |
|---|---|---|
| BARU | `src/portal/PortalContext.tsx` | provider + hook `usePortal` |
| BARU | `src/portal/EndpointSwitcher.tsx` | switcher + badge header |
| SENTUH | `src/layouts/AdminLayout.tsx` | sisip switcher di header kanan |
| SENTUH | `src/pages/Portal.tsx` | hanya spacing kosmetik |
| DILARANG | `src/api/client.ts`, `src/App.tsx`, halaman admin lain, seluruh folder Go | |

## Kontrak data (terverifikasi dari kode, jangan diubah)

- `GET /api/admin/endpoints` via `apiLocal` → `{ items: [{ key, label,
  base_url, status, is_local }] }` (`Portal.tsx:48-58`).
- `POST /api/admin/portal-token` body `{ audience: <key> }` via `apiLocal` →
  `{ portal_token }` (`Portal.tsx:83-96`).
- `GET /api/business/whoami` via `apiPortal` → 200 `{ instance: <key>, ... }`
  | 403 (belum approve / dinonaktifkan) (`Portal.tsx:98-107`).
- Helper: `setPortalSession(token, key)` mengisi `portal_token` +
  `active_endpoint`; `clearPortalSession()` membuang keduanya dan `main_token`
  UTUH (`client.ts:86-95`).
- Interceptor `apiPortal`/`apiClient` otomatis mengirim `X-Portal-Token` +
  `X-Branch-Endpoint` — JANGAN set header manual di komponen.

## Tugas

### T-1 `src/portal/PortalContext.tsx`

- State: `endpoints: EndpointView[]`, `status: 'idle' | 'loading' | 'active' |
  'pending' | 'error'`, `whoami`, `message: string`.
- `loadEndpoints()` di-mount provider (provider di-mount dalam
  `<AdminLayout>` yang sudah di bawah `<RequireAuth>`); guard: skip bila
  `!getMainToken()`.
- `enterPortal(key)`:
  1. jika `key === getActiveEndpoint()` → no-op;
  2. `status='loading'` → `POST portal-token {audience: key}` →
     `setPortalSession(portal_token, key)`;
  3. probe 1× `GET /api/business/whoami`: 200 → `status='active'`;
     403 → `status='pending'` (pesan dari body error; 403 + body memuat
     "pending" = menunggu persetujuan, selain itu `status='error'`);
  4. gagal token → `status='error'` + toast.
- `refreshStatus()`: probe ulang whoami (dipakai setelah admin cabang
  menyetujui enrollment).
- `exitPortal()`: `clearPortalSession()` + reset state ke `'idle'`.
- Baris `is_local: true` BUKAN pilihan masuk portal — "Pusat" adalah kondisi
  tanpa sesi portal (map ke `exitPortal`).

### T-2 `src/portal/EndpointSwitcher.tsx`

- Dipasang di sisi KANAN header `AdminLayout` (header
  `AdminLayout.tsx:148-168` kini `justify-between` dengan hanya anak kiri:
  hamburger + judul; sisi kanan kosong).
- Badge default: `Pusat — BBH Data Board`; saat sesi portal:
  `<label> — cabang`.
- Warna badge: `active` hijau, `pending` kuning ("Menunggu persetujuan admin
  cabang"), `error` merah.
- Isi dropdown: daftar cabang (`key — label`), aksi **Cek Status**
  (`refreshStatus`, menampilkan hasil whoami terakhir), tombol **Kembali ke
  Pusat** (hanya saat sesi portal aktif).
- `data-testid` wajib: `endpoint-switcher`, `active-endpoint-badge`,
  `portal-exit`, `portal-status` (paritas dengan `data-testid` lama di
  `Portal.tsx` untuk e2e).

### T-3 `AdminLayout.tsx`

- Import + render `<EndpointSwitcher />` di sisi kanan header. Tanpa
  perubahan lain (sidebar, menu, footer tidak disentuh).

### T-4 `Portal.tsx` (kosmetik saja)

- Bereskan label/tombol menempel (`active_endpointKOSONG`,
  `Masuk PortalPanggil WhoAmI`) dengan gap/margin. Logika, data-testid, dan
  teks log tidak berubah.

## Acceptance criteria

- **AC-1** Setelah login pusat, switcher di header menampilkan semua endpoint
  non-local (jogja/semarang/lombok/java-children) tanpa reload halaman.
- **AC-2** Memilih cabang → `portal_token` & `active_endpoint` terisi
  (terverifikasi lewat snapshot `/portal`), badge berubah ke label cabang.
- **AC-3** Request `/api/business/*` dari UI membawa `X-Portal-Token` +
  `X-Branch-Endpoint` (bukti: devtools Network atau log cabang).
- **AC-4** User pusat yang belum enroll → probe 403 → badge kuning "Menunggu
  persetujuan admin cabang" (bukan error mentah); setelah admin cabang
  menyetujui (via API cabang — UI approve belum ada) + klik **Cek Status** →
  200 → badge hijau.
- **AC-5** "Kembali ke Pusat" → dua key localStorage terhapus, `main_token`
  utuh, header kembali "Pusat".
- **AC-6** `/portal` berfungsi identik (snapshot + log utuh) — regresi nol;
  hanya tampilan dirapikan.
- **AC-7** `npm run build` (tsc -b && vite build) lulus tanpa error TS.
- **AC-8** Tidak ada file di luar daftar File yang berubah
  (`git status --porcelain` hanya memuat 4 path di atas + `dist/`).

## Urutan kerja & bukti (checkpoint)

| CP | Isi | Bukti yang dicatat |
|---|---|---|
| CP-1 | T-1 selesai, build lulus | output build |
| CP-2 | T-2 + T-3: switcher hidup, enter portal, badge | screenshot state `active` |
| CP-3 | Alur pending→approve→active (pakai user pusat yang belum enroll, mis. `ops@bbh.dev`) | screenshot pending & active |
| CP-4 | T-4 + exit portal + build final + checklist AC | daftar AC lulus + `git status --porcelain` |

## Verifikasi manual (siap copy-paste)

1. `cd sub_app/webapp && npm run build` (host).
2. Login pusat (`root@bbh.dev` atau `root@admin.local`) di `/app/login` →
   header menampilkan switcher (AC-1).
3. Pilih `jogja` → badge aktif → buka `/app/portal`: `portal_token=ADA`,
   `active_endpoint=jogja` (AC-2).
4. **Cek Status** → 200 (user lama ter-enroll) atau 403 pending (user baru)
   → setujui enrollment di cabang (`GET /api/admin/sso-enrollments` →
   `POST /api/admin/sso-enrollments/:id/approve` memakai JWT
   `admin@jogja.local`) → **Cek Status** lagi → 200 (AC-4).
5. **Kembali ke Pusat** → `/app/portal`: `portal_token=KOSONG`,
   `active_endpoint=KOSONG` (AC-5).
6. Verifikasi header portal-token di Network tab pada request
   `/api/business/whoami` (AC-3).

## Catatan operasional (di luar scope dev — jangan dieksekusi oleh dev)

- Frontend = `dist/` statis dari disk (bind-mount): setelah build cukup
  refresh browser; **tidak perlu** restart container/server.
- Deployment lintas instance tidak diperlukan: fitur hanya UI pusat; cabang
  tidak butuh build baru (approve enrollment tetap via API cabang).

## Catatan Implementasi (Dev)

Bukti & alasan per berkas. Semua perubahan frontend-only; tidak ada perubahan
Go/migrasi/api-tests dan tidak ada dependensi npm baru (AC-8).

1. `sub_app/webapp/src/portal/PortalContext.tsx:141` — `PortalProvider`
   memasang state machine portal (`idle|loading|active|pending|error`) tepat di
   bawah `RequireAuth`, sehingga semua aksi token terpusat di satu tempat (T-1).
2. `PortalContext.tsx:146` — status awal `loading` bila `getActiveEndpoint()`
   dan `getPortalToken()` sudah ada. Tanpa ini badge sempat berkedip
   "Pusat — BBH Data Board" setelah reload sebelum probe selesai.
3. `PortalContext.tsx:173` — `probe()` hanya memanggil
   `GET /api/business/whoami` lewat `apiPortal`, jadi header `X-Portal-Token`
   dan `X-Branch-Endpoint` diurus `client.ts` (tidak ada header manual) — AC-3.
4. `PortalContext.tsx:184` — 403 dengan body `pending` dipetakan ke status
   `pending` + `PENDING_MESSAGE`, bukan dianggap error, agar AC-4 tampil kuning
   tanpa toast merah.
5. `PortalContext.tsx:196` — `enterPortal()` meminta token ke
   `POST /api/admin/portal-token {audience}` lalu menyimpannya lewat
   `setPortalSession()`; storage tidak disentuh bila permintaan gagal (T-1).
6. `PortalContext.tsx:230` — `refreshStatus()` dipakai tombol "Cek Status";
   tanpa sesi ia mereset ke `idle`, sehingga tidak ada probe tanpa token.
7. `PortalContext.tsx:244` — `exitPortal()` memakai `clearPortalSession()`
   (`portal_token` + `active_endpoint` dibuang, `main_token` utuh = AC-5) dan
   menaikkan `requestRef` untuk membatalkan probe yang masih berjalan.
8. `PortalContext.tsx:255` — effect mount memuat daftar cabang sekali dan
   memulihkan sesi tersimpan dengan satu `probe`, sehingga reload tidak
   memaksa user memilih cabang ulang.
9. `PortalContext.tsx:267` — `branches` menyaring `is_local`, jadi instance
   `local` tidak muncul sebagai pilihan cabang (AC-1).
10. `sub_app/webapp/src/portal/EndpointSwitcher.tsx:91` — badge tombol dengan
    `data-testid="endpoint-switcher"` + `aria-haspopup="menu"` supaya switcher
    dapat diakses keyboard dan mudah diverifikasi otomatis (T-2, AC-1).
11. `EndpointSwitcher.tsx:18` — peta warna per status (`active` hijau,
    `pending` kuning, `error` merah, `loading/idle` netral) memberi umpan balik
    visual status kepesertaan tanpa membuka halaman terpisah.
12. `EndpointSwitcher.tsx:115` — teks badge (`active-endpoint-badge`)
    "Pusat — BBH Data Board" / "<label> — cabang" agar instance aktif selalu
    terbaca di header (D-13/D-14).
13. `EndpointSwitcher.tsx:126` — pesan status (`portal-status`, `role="status"`)
    memuat teks baku pending "Menunggu persetujuan admin cabang" (AC-4).
14. `EndpointSwitcher.tsx:151` — daftar cabang dalam `max-h-80 overflow-y-auto`
    supaya tetap ringkas bila jumlah cabang bertambah.
15. `EndpointSwitcher.tsx:174` — setiap item memakai ikon `Building2`, baris
    kedua `key · base_url`, dan `Check` pada cabang aktif; item aktif
    di-`disabled` kecuali status `error` agar user bisa mencoba ulang.
16. `EndpointSwitcher.tsx:220` — "Kembali ke Pusat" (`portal-exit`) didelegasikan
    ke `exitPortal()`, bukan menghapus storage langsung (batas WO).
17. `sub_app/webapp/src/layouts/AdminLayout.tsx:60` — `AdminLayout` membungkus
    body lama (dipindah utuh ke `AdminLayoutContent`) dengan `PortalProvider`,
    sehingga provider hidup di dalam shell admin dan state portal tidak bocor
    ke halaman publik (T-3).
18. `AdminLayout.tsx:184` — `<EndpointSwitcher />` ditempatkan di kanan header
    (setelah judul + ikon), tanpa mengubah sidebar/menu/footer.
19. `sub_app/webapp/src/pages/Portal.tsx:140` — halaman debug harness dirapikan
    memakai `Card`/`Button`/`Badge`/`Input` (container `max-w-5xl`) agar tidak
    lagi tampil sebagai HTML polos; logika, `data-testid`, dan teks log tidak
    berubah sehingga AC-6 regresi nol.
20. `Portal.tsx:290` — snapshot tetap satu blok `data-testid="snapshot"`
    berisi `value-main_token`, `value-portal_token`, `value-active_endpoint`
    (dipakai verifikasi AC-2/AC-5); `data-testid="log"` di `Portal.tsx:325`
    tetap `<pre>` berisi baris log apa adanya.

### Hasil verifikasi (lingkungan Tailscale)

Base URL: `http://100.125.175.34:8082/app` (pusat), cabang `:8101` jogja,
`:8102` semarang, `:8103` lombok, `:8104` java-children.

- **AC-1 LULUS** — login `root@admin.local`; dropdown memuat 4 cabang non-local
  (Java Children, Jogja, Lombok, Semarang) tanpa reload.
- **AC-2 LULUS** — pilih Jogja → badge "Jogja Baby Home — cabang";
  `/app/portal` snapshot: `main_token=ADA`, `portal_token=ADA`,
  `active_endpoint=jogja`.
- **AC-3 LULUS** — `GET /api/business/whoami` membawa
  `X-Portal-Token` (287 char) dan `X-Branch-Endpoint: jogja`.
- **AC-5 LULUS** — "Kembali ke Pusat" → badge "Pusat — BBH Data Board";
  snapshot `portal_token=KOSONG`, `active_endpoint=KOSONG`, `main_token=ADA`.
- **AC-6 LULUS** — alur `/app/portal` (login → masuk portal → whoami) tetap
  menghasilkan log identik: `endpoints: 5`, `login OK (main_token disimpan)`,
  `masuk portal "jogja" OK (portal_token disimpan)`, `whoami OK: {...}`.
- **AC-7 LULUS** — `npm run build` sukses (tsc -b + vite build, ±21s).
- **AC-8 LULUS** — `git status --porcelain` hanya 3 entri whitelist
  (`AdminLayout.tsx`, `Portal.tsx`, folder `src/portal/`).
- **AC-4 BELUM DIVERIFIKASI** — butuh akun pusat yang belum ter-enroll di
  cabang (mis. `ops@bbh.dev`); kredensial tidak tersedia di
  `.credentials/`. Menunggu keputusan owner.
