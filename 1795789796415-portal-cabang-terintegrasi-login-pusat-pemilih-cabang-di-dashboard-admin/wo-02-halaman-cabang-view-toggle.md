---
title: 'WO-02 — Halaman Cabang: tampil + enable/disable (frontend-only)'
description: 'Halaman /endpoints di shell admin: daftar cabang (read-only) + toggle status active/suspended; TANPA create/edit/delete'
status: complete
created: "2026-11-27"
updated: "2026-11-27"
---
# WO-02 — Halaman Cabang: tampil + enable/disable (frontend-only)

## Referensi

- `konteks.md` & `wo-01-integrasi-portal-dashboard.md` (topik ini) — konvensi
  sudah disepakati (helper `client.ts`, data-testid, guard per halaman);
  **ikuti pola yang dipakai WO-01**.
- Backend **SUDAH LENGKAP, JANGAN DIUBAH**: `plugins/endpoint/plugin.go:43-58`
  (routes + RBAC) dan `plugins/endpoint/services/service.go` (semantik status).
- Keputusan owner (2026-11-27): halaman **hanya tampil + enable/disable** —
  **TANPA create, edit, atau delete** di UI.

## Prinsip batas (HARD CONSTRAINT)

1. **Frontend-only.** Tidak ada perubahan folder Go/migrasi/api-tests; tidak
   ada dependensi npm baru.
2. **Tanpa UI create/edit/delete** — hanya daftar + toggle status. API
   `POST/PUT/DELETE /api/admin/endpoints` TIDAK dipanggil dari halaman ini.
3. **Tidak mengubah `src/api/client.ts`** — pakai `apiClient` (route
   `/api/admin/*` otomatis jalur lokal, D-14).
4. **Whitelist file (hanya 3):**
   - BARU `src/pages/admin/endpoints/EndpointListPage.tsx`
   - SENTUH `src/App.tsx` — tambah 1 route
   - SENTUH `src/navigation/menu.ts` — tambah 1 item menu (path aktual di repo; WO semula menulis `menu.tsx`)
5. Guard izin mengikuti pola halaman lain: lihat `endpoint.read`, toggle hanya
   tampil aktif bila `endpoint.write` (pola `useAuth().hasPermission` seperti
   di halaman hsh/sec/warehouse).
6. `npm run build` wajib hijau; gaya kode mengikuti halaman list existing
   (rujukan: `pages/admin/warehouse/ProductListPage.tsx`).

## Kontrak data (terverifikasi dari kode — jangan diubah)

- `GET /api/admin/endpoints` → `{ items: [{ key, label, base_url, purpose,
  auth_kind, timeout_ms, status, is_local, is_active }] }`
  ⚠️ **`is_active` = penanda "sedang dipakai" (hanya satu true), BUKAN flag
  enable/disable.** Flag on/off adalah kolom `status`.
- `PATCH /api/admin/endpoints/{key}` body `{"status": "active" | "suspended"}`
  → 200 `{...view}`. Nilai lain ditolak backend (`validateStatus`,
  `service.go:120-125`).
- **Backend menolak men-suspend endpoint yang sedang dipakai**
  (`service.go:297-299` → error) — UI harus menampilkan pesan ini dengan sopan
  dan membiarkan status tak berubah.
- Efek bisnis `suspended`: pemakaian endpoint ditolak
  `ErrEndpointSuspended` — "endpoint is suspended: activate its status before
  using it" (`service.go:32,363`) — dipakai sebagai verifikasi E2E di AC-3.
- Proteksi UX di sisi UI (tanpa mengubah backend): baris `is_local: true`
  tidak diberi toggle.

## Tugas

### T-1 `src/pages/admin/endpoints/EndpointListPage.tsx` (baru)

- Tabel kolom: Key, Label, Base URL, Status (badge hijau `active` / abu-abu
  `suspended`), Dipakai (✅ bila `is_active` atau `is_local`), Aksi (toggle).
- Toggle (switch) per baris cabang (`is_local: false`):
  - status `active` → aksi "Nonaktifkan" → `PATCH {status:"suspended"}`
  - status `suspended` → aksi "Aktifkan" → `PATCH {status:"active"}`
  - baris yang sedang dipakai (`is_active: true`): toggle disabled + tooltip
    "Tidak bisa dinonaktifkan: sedang dipakai"
  - gagal (mis. backend menolak) → toast error dengan pesan dari body,
    status TIDAK berubah di UI (refetch untuk kepastian)
- Tanpa tombol tambah/ubah/hapus; tanpa form.
- `data-testid` wajib: `endpoint-page`, `endpoint-row-{key}`,
  `endpoint-status-{key}`, `endpoint-toggle-{key}`.

### T-2 `src/App.tsx` (1 route)

- `<Route path="/endpoints" element={<EndpointListPage />} />` di dalam grup
  admin ter-guard, konsisten urutan import/route yang ada.

### T-3 `src/navigation/menu.ts` (1 item)

- Tambah item "Cabang" → `/endpoints` pada group yang relevan — ikuti struktur
  `menu.ts` yang ada (tipe NavItem/NavGroup + mapping ikon yang dipakai
  `AdminLayout.iconFor`); judul header otomatis mengikuti jika dipetakan.

## Acceptance criteria

- **AC-1** `/endpoints` menampilkan semua endpoint (local + 4 cabang) dengan
  kolom lengkap dan badge status yang benar.
- **AC-2** Toggle "Nonaktifkan" pada cabang (mis. `lombok`) → badge berubah
  `suspended`; "Aktifkan" kembali → `active` (refetch membuktikan).
- **AC-3** Efek bisnis terbukti: saat `suspended`, `POST /api/admin/portal-token
  {audience: <key>}` di pusat gagal dengan pesan "endpoint is suspended";
  setelah diaktifkan kembali, portal-token sukses lagi.
- **AC-4** Toggle pada endpoint yang sedang dipakai disabled (tooltip); jika
  dipaksa via API tampil pesan error backend dengan sopan, status tetap.
- **AC-5** Tidak ada UI create/edit/delete di halaman (cek visual + kode).
- **AC-6** Tanpa izin `endpoint.write` toggle tidak tampil; tanpa
  `endpoint.read` halaman menampilkan pesan akses ditolak (pola existing).
- **AC-7** `npm run build` hijau; `git status --porcelain` hanya berisi 3 file
  whitelist (+`dist/`).
- **AC-8** Regresi nol: WO-01 (switcher/badge) & `/portal` tetap berfungsi.

## Urutan kerja & bukti (checkpoint)

| CP | Isi | Bukti |
|---|---|---|
| CP-1 | T-2 + T-3 + kerangka T-1 (list tampil) | output build + screenshot |
| CP-2 | Toggle suspend/aktifkan + verifikasi E2E AC-3 | screenshot badge + output portal-token gagal/sukses |
| CP-3 | Guard izin + checklist AC | daftar AC lulus + `git status --porcelain` |

Commit per CP: `feat(endpoints): CP-x ...` (lokal, JANGAN push).

## Verifikasi manual (siap copy)

1. `cd sub_app/webapp && npm run build` (host) → refresh browser.
2. Login pusat → menu **Cabang** → tabel 5 baris terlihat (AC-1).
3. Nonaktifkan `lombok` → badge abu; cek `POST /api/admin/portal-token
   {"audience":"lombok"}` (JWT admin) → error "endpoint is suspended" (AC-3).
4. Aktifkan kembali → portal-token sukses (AC-2/AC-3).
5. Coba nonaktifkan endpoint yang sedang dipakai → toggle disabled (AC-4).
6. Pastikan halaman lain & switcher WO-01 tidak terpengaruh (AC-8).

## Catatan operasional (di luar scope dev)

- Frontend = `dist/` dari disk: selesai build cukup refresh browser, tanpa
  restart container.
- Status `suspended` adalah data DB — tidak tersentuh update binary cabang.

---

## Bukti pelaksanaan (2026-11-27, selesai)

Base URL uji: `http://100.125.175.34:8082` (IP Tailscale, bukan localhost).
Build: `npm run build` hijau di setiap CP (CP-1 28.29s, CP-2 30.49s, CP-3 29.05s).

### Commit per CP (lokal, TIDAK di-push)

| CP | Commit | Isi |
|---|---|---|
| CP-1 | `e87de84` | route `/endpoints` + menu "Cabang" + daftar read-only |
| CP-2 | `ea9832c` | toggle aktif/nonaktif (`PATCH {status}`) + toast + refetch |
| CP-3 | `773b6eb` | guard izin `endpoint.read` / `endpoint.write` |

### Checklist AC

- **AC-1 LULUS** — `/endpoints` menampilkan 5 baris: `local`, `java-children`,
  `jogja`, `lombok`, `semarang`; kolom Key/Label/Base URL/Status/Dipakai/Aksi
  lengkap; badge `active` hijau untuk semua; `local` → "✅ pusat" dan Aksi "—".
  Judul dokumen "Cabang · BBH Data Board — Admin"; menu "Cabang" di grup
  Administrasi.
- **AC-2 LULUS** — klik "Nonaktifkan" pada `lombok` → toast "Cabang Lombok Baby
  Home dinonaktifkan.", badge jadi `suspended`, label tombol jadi "Aktifkan";
  klik "Aktifkan" → toast "Cabang Lombok Baby Home diaktifkan.", badge
  `active` kembali. Registry via API ikut berubah (`lombok suspended` →
  `lombok active`).
- **AC-3 LULUS (dengan catatan semantik)** — saat `lombok` `suspended`:
  `GET /api/business/whoami` dengan `X-Portal-Token` + `X-Branch-Endpoint:
  lombok` → **409** `{"error":"endpoint \"lombok\" berstatus suspended;
  aktifkan dulu sebelum dipakai"}`. Setelah diaktifkan kembali → **200**
  `{"instance":"lombok",...}`.
  ⚠️ Catatan: `POST /api/admin/portal-token {audience:"lombok"}` tetap **200**
  meski endpoint `suspended` — penegakan status terjadi saat **pemakaian**
  endpoint, bukan saat penerbitan token. Ini berbeda dari kalimat AC yang
  menyebut portal-token gagal "endpoint is suspended". Perilaku backend tidak
  diubah (di luar scope); AC dibuktikan lewat jalur pemakaian yang sebenarnya.
- **AC-4 LULUS** — dengan `semarang` sebagai endpoint "sedang dipakai"
  (`is_active: true`), tombol toggle-nya `disabled` dengan tooltip
  **"Tidak bisa dinonaktifkan: sedang dipakai"** (4 toggle lain normal).
  Paksa via API: `PATCH /api/admin/endpoints/local {status:"suspended"}` → **409**
  `refusing to suspend the active endpoint: switch to another endpoint first`;
  status tetap `active` (UI juga menampilkan pesan sopan lewat
  `toggleErrorMessage`). Screenshot: `assets/wo02-ac4-toggle-disabled.png`.
- **AC-5 LULUS** — halaman hanya memanggil `GET /api/admin/endpoints` dan
  `PATCH /api/admin/endpoints/{key} {status}`; tidak ada `POST`/`PUT`/`DELETE`,
  tidak ada form/modal/tombol Tambah-Ubah-Hapus (grep kode: hanya komentar).
- **AC-6 LULUS** — akun `viewer` (punya `endpoint.read`, tanpa
  `endpoint.write`): 5 baris tampil, **0 toggle** (kolom Aksi "—") —
  `assets/wo02-ac6-viewer-tanpa-toggle.png`. Akun tanpa grup (tanpa
  `endpoint.read`): halaman menampilkan **"Akses ditolak (403) — Halaman ini
  membutuhkan izin `endpoint.read`"**, tanpa memanggil API (fail-closed) —
  `assets/wo02-ac6-tanpa-read-403.png`. Akun uji dihapus kembali setelah
  verifikasi.
- **AC-7 LULUS** — build hijau; `git status --porcelain` hanya
  `M sub_app/webapp/src/pages/admin/endpoints/EndpointListPage.tsx` sebelum
  commit CP-3 (tidak ada file di luar whitelist).
- **AC-8 LULUS** — switcher WO-01 tetap ada di `/app/dashboard` (badge
  "Pusat — BBH Data Board"), menu "Cabang" muncul, dan `/app/portal` tetap
  render penuh (8506 char, tanpa console error). Catatan: URL `/portal` tanpa
  prefix `/app` memang tidak pernah render karena `BrowserRouter basename="/app"`
  di `src/main.tsx` — kondisi ini sudah ada sebelum WO-02 (bukan regresi).

### File yang diubah

1. BARU `sub_app/webapp/src/pages/admin/endpoints/EndpointListPage.tsx`
2. `sub_app/webapp/src/App.tsx` — 1 route `/endpoints`
3. `sub_app/webapp/src/navigation/menu.ts` — 1 item menu "Cabang"

Tidak ada perubahan backend/Go/migrasi/api-tests; tidak ada dependensi npm baru.
