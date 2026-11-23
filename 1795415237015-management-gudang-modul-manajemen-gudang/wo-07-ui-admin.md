---
title: WO-07 — UI admin /warehouse
description: "SPA: daftar item, batch, masuk/keluar, summary"
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---

# WO-07 — UI admin `/warehouse/*`

## Prasyarat

WO-03 lulus (bisa paralel dgn WO-04..06).

## Lingkup

- `src/api/wh.ts`, `src/lib/whFields.ts` (mirror fields.yaml + drift check), `src/pages/admin/warehouse/*`.
- Halaman: kategori & produk (CRUD + cari), gudang (CRUD), daftar item + stok terkini + badge near-expiry, detail item = daftar batch (umur hari, expiry, terlama dulu) + form masuk + modal keluar (pilih batch, reason/destination), summary/unduh template/impor.
- Rute `/warehouse/*` di `App.tsx`; grup nav "Gudang" di `src/navigation/menu.ts` filter `warehouse.read`.
- PermissionGate level halaman; tombol mutasi `hasPermission`; Pola komponen `src/components/ui/*`.

## Acceptance Criteria

- AC-1 `tsc -b && vite build` hijau; dist bebas URL backend (grep host:port kosong).
- AC-2 E2E Playwright via dev server :5173 (Tailscale): CRUD produk, batch masuk, keluar kurang dari sisa → pesan 409, near-expiry badge.
- AC-3 RBAC viewer: tombol mutasi tersembunyi; admin tanpa `warehouse.read` → guard 403.
- AC-4 Gotcha UI table: `TableSkeletonRows({colSpan,rows})`/`TableStateRow({colSpan,empty,...})` — tanpa prop `columns`/children.

## Catatan Implementasi (Dev)

Referensi: D-1..D-8 & O-1/O-2 (`spec.md` §8), kontrak WO-03 (batch/movement/summary) & WO-04..06 (template/importer). Semua nomor baris merujuk kondisi saat WO ini selesai.

### API client & mirror skema

- `src/api/wh.ts` — satu pintu semua endpoint `/api/admin/wh/*` + 2 route flat `/api/admin/wh-template*`. Tipe ditulis manual mengikuti struct Go (`services.BatchView`, `Summary`, `ImportResult`, `ImportBatchReport`), **bukan** `any`; payload unggah memakai `FormData` supaya boundary diurus browser.
- `src/api/wh.ts:349` `filenameFrom()` + `:356` `saveBlob()` — unduhan dibuat dari `blob:` URL + `Content-Disposition`, **bukan** URL absolut. Ini yang menjaga AC-1 (dist bebas `host:port`): tidak ada `http://…` yang perlu di-`grep` hilangkan.
- `src/api/wh.ts:368` `downloadTemplate()` — `responseType: 'blob'`, `params.format`; gagal → lempar ke pemanggil (halaman menampilkan `Alert`), tidak menelan error.
- `src/lib/whFields.ts` — mirror `plugins/warehouse/schema/fields.yaml` (enum `wh_direction` 2 nilai, 9 satuan, 8 kolom template DATA) + **resep drift-check** di header komentar (`sed -n` per blok). Dipisah dari `lib/fields.ts`/`hshFields.ts`/`secFields.ts` karena beda sumber kebenaran.
- `src/lib/whFields.ts:35` `WH_NEAR_EXPIRY_DAYS = 30` hanya untuk **teks bantuan**; angka badge selalu dari field server (`near_expiry`, `near_expiry_count`) — UI tidak menghitung ulang selisih tanggal (hindari beda zona waktu dengan server).

### Halaman & komponen

- `PermissionGate.tsx:21` — guard level halaman: `hasPermission(permission)` dari `AuthContext` (satu sumber logika izin, pola `sec`/`hsh`). Gagal → `Alert tone="danger"` "Akses ditolak (403)" + kode izin yang dibutuhkan, **bukan** redirect diam-diam (pengguna tahu harus minta izin apa).
- `CategoryListPage.tsx:36-38` (idem `ProductListPage.tsx:41-43`, `StockListPage.tsx:35-36`, `StockDetailPage.tsx:56-57`) — tiga tingkat izin terpisah: `canWrite` (`warehouse.write`) mengatur tombol tambah/ubah, `canDelete` (`warehouse.delete`, root-only di backend) mengatur tombol hapus. Viewer default hanya dapat `warehouse.read` → tombol mutasi **tidak dirender** (AC-3).
- `CategoryListPage.tsx:146` / `ProductListPage.tsx:204` / `StockListPage.tsx:134` / `StockDetailPage.tsx:208,314` — `TableSkeletonRows({colSpan, rows})` untuk loading dan `TableStateRow({colSpan, empty|error, emptyMessage, onRetry})` untuk kosong/error, sesuai AC-4: **tidak ada** prop `columns`, **tidak** menerima children. `colSpan` = jumlah kolom tabel yang bersangkutan (dihitung manual per halaman: 3 / 5 / 6 / 7 / 5).
- `ConfirmDeleteModal.tsx:40` — `statusOf(err) === 409` → pesan "Data masih dipakai baris lain, jadi belum bisa dihapus." Backend menolak hapus kategori/produk yang masih dipakai dan gudang yang masih punya batch; UI menerjemahkan kode, bukan menebak teks server.
- `CategoryFormModal.tsx:61` & `ItemFormModal.tsx:82` — 409 = duplikat nama (unik per instance untuk kategori, unik per kategori untuk produk) → pesan spesifik di dalam modal + form tetap terisi supaya pengguna bisa perbaiki nama, bukan modal tertutup dan data hilang.
- `ItemFormModal.tsx` — dropdown kategori hanya berisi kategori **aktif** (plus kategori item itu sendiri saat mengubah), karena backend menolak `category_id` nonaktif pada item baru. Satuan memakai `WH_UNITS` (9 nilai) — daftar REFERENSI yang sama dengan dropdown template impor, jadi tidak ada satuan "karangan" di master yang akan ditolak importer.
- `WarehouseFormModal.tsx` — hanya `name` + `note`; **tanpa** toggle aktif karena tabel `wh_warehouses` tidak punya kolom `is_active` (jangan bikin UI yang menjanjikan field tak ada). Ditambah keterangan O-2: gudang tidak dibuat otomatis dari berkas impor.
- `MovementModal.tsx:78` — `statusOf(err) === 409` → "Stok tidak cukup. Sisa batch ini hanya `<qty> <unit>`". Angka sisa diambil dari `batch.qty_remaining` yang **sudah dimuat server**, bukan dari state lokal, sehingga pesan selalu konsisten dengan penolakan backend (WO-03: `ErrInsufficientStock`).
- `MovementModal.tsx` — validasi klien mengikuti kontrak server: `OUT` jumlah > 0; `ADJUST` jumlah ≠ 0 (positif menambah, negatif mengurangi) **dan** `reason` wajib. Label/hint berubah mengikuti arah ("Jumlah keluar" vs "Selisih penyesuaian"), supaya pengguna tidak mengira penyesuaian mengurangi seperti `OUT`.
- `StockListPage.tsx:31` & `StockDetailPage.tsx:50` — komentar eksplisit: **stok tidak pernah disimpan/dijumlahkan di UI**, semua dari `GET /wh/summary` (`SUM(qty_remaining)` di server) atau `qty_remaining` batch. Ini keputusan konsistensi: satu sumber angka, tidak ada selisih akibat dua perhitungan.
- `StockDetailPage.tsx:109` `totalRemaining` — **satu-satunya** penjumlahan di klien, murni presentasi untuk header "Stok saat ini"; bukan cache, selalu diturunkan ulang dari daftar batch yang baru dimuat. Diberi komentar agar tidak dipakai sebagai sumber kebenaran.
- `StockDetailPage.tsx` — daftar batch `limit: 200` urut terlama dulu (sesuai kebutuhan FEFO), tanpa paginasi UI: batch per produk biasanya sedikit dan justru urutan umur yang penting. Batch `qty_remaining <= 0` → tombol "Keluar" dinonaktifkan + tooltip, karena backend akan menolak movement pada batch habis.
- `shared.tsx:47` `NEAR_EXPIRY_HINT` & `ExpiryBadge` — 3 keadaan: tanpa kadaluarsa (neutral), `near_expiry` true (danger "Segera kadaluarsa"), masih jauh (info "Aman"). Ambang dihitung server, jadi kalau `services.NearExpiryDays` berubah UI ikut otomatis.
- `TemplatePage.tsx` — 4 seksi linear (unduh → uji/impor → laporan → audit). `canWrite` false hanya mengganti **seksi impor** dengan `Alert` ("Butuh izin `warehouse.write`"), sedangkan unduh + uji + muat audit tetap jalan (semuanya guard read) — jadi viewer tetap bisa memeriksa template tanpa tombol mati yang membingungkan.
- `TemplatePage.tsx:32-52` — peta `ACTION_TONES`/`ACTION_LABELS`/`BATCH_STATUS_LABELS` menerjemahkan `created|updated|skipped|rejected|pending` & `staged|committed|failed` ke Badge Indonesia. `rejected` selalu `danger`, `pending` `neutral` (baris uji kering = belum ada aksi tulis).
- `TemplatePage.tsx` — `newMasters` getter menangani **dua bentuk**: `result.new_masters` (array, WO-06 import) dan `audit.batch.new_masters` (JSON string di DB, hasil `getImportBatch`) — tanpa normalisasi ini seksi "Master baru dibuat otomatis" kosong saat memuat audit.
- `TemplatePage.tsx` — toast akhir impor memakai tone `error` bila ada baris ditolak, `success` bila bersih. **Gotcha**: `ToastTone` hanya `'info' | 'success' | 'error'` (`components/ui/Toast.tsx`) — tidak ada `'warning'`; sempat gagal `tsc` (TS2345) dan diperbaiki ke `'error'`.
- `TemplatePage.tsx` — input ID batch otomatis terisi `result.batch_id` setelah impor, sehingga alur "impor → Muat audit batch" tidak perlu salin-tempel manual.

### Routing, navigasi, batasan

- `src/App.tsx:113-118` — semua segmen **statis** (`/warehouse/categories|products|warehouses|template|stock`) didaftarkan **sebelum** `/warehouse/stock/:id`. Urutan ini wajib: `react-router` memilih rute pertama yang cocok, jadi `:id` di depan akan menelan `/warehouse/stock` dan menampilkan halaman detail dengan id kosong.
- `src/navigation/menu.ts:94-98` — grup nav "Gudang" berisi 5 item, semuanya `permission: 'warehouse.read'`. Tanpa izin itu grup **hilang seluruhnya** dari sidebar (difilter oleh `AdminLayout`), bukan tampil lalu 403 — guard halaman tetap ada sebagai lapisan kedua untuk akses via URL langsung.
- `src/navigation/menu.ts:97` — label item "Gudang Fisik" (bukan "Gudang") untuk menghindari "Gudang › Gudang" yang ambigu.
- **Tidak menyentuh** `src/layouts/AdminLayout.tsx` dan `src/styles/index.css` (keduanya dipakai patch ingest di WO lain). Batasan lebar konten karena itu diberikan per halaman via `max-w-*` di dalam halaman, bukan di layout.
- Format tanggal: kolom tanggal (`entered_at`, `expires_at`, `Tanggal masuk` di laporan) memakai `YYYY-MM-DD` apa adanya dari server — **berbeda** dari domain `hsh`/`sec` yang memakai `DD/MM/YYYY` pada berkas impor. Kebetulan tanggal (`formatDate` di `lib/format.ts`) memakai `timeZone: 'UTC'` agar `2026-11-23` tidak bergeser sehari di zona +07.

### Verifikasi (AC-1..AC-4)

- AC-1: `npx tsc -b` bersih, `npm run build` hijau; `grep -rEo "http://(100\.[0-9.]+|localhost):(8082|5173)" dist` → **kosong**.
- AC-2 (Playwright, dev `:5173` via Tailscale): kategori CRUD + duplikat 409 tampil di modal; produk CRUD + 409 "Produk dengan nama itu sudah ada di kategori ini."; gudang CRUD; batch masuk 20 box `expires_at=2026-12-10` → badge "Segera kadaluarsa" (sisa 30 hari) muncul di list & detail; keluar 999 → 409 "Stok tidak cukup. Sisa batch ini hanya 20 box."; keluar 5 → sisa 15; penyesuaian 0 → validasi klien; penyesuaian +3 → sisa 18 + riwayat `+3 box`; unduh template XLSX (10.180 B) & CSV (89 B) 200 dengan `Content-Disposition`; uji kering 3 baris → semua `pending`, master 0; impor → **2 dibuat / 1 ditolak (gudang asing, HTTP 200)** + `new_masters` 3 entri; `Muat audit batch` menampilkan 3 baris `created/rejected`; `Uji & unduh umpan balik` → `X-Validate-Total: 3`, `X-Validate-Valid: 3`, `X-Validate-Rejected: 0`.
- AC-3: `whviewer` (grup `viewer`, hanya `warehouse.read`) → halaman terbuka, kolom Aksi kosong, tombol "Tambah", "Ubah", "Hapus", "Barang masuk", "Keluar" **tidak ada**, seksi impor diganti `Alert` "Butuh izin `warehouse.write`". `whnoright` (`--no-default-group`) → sidebar hanya "Dashboard" dan `/warehouse/stock` menampilkan "Akses ditolak (403) / membutuhkan izin `warehouse.read`".
- AC-4: seluruh tabel memakai `TableSkeletonRows colSpan` / `TableStateRow colSpan`; tidak ada pemakaian prop `columns` atau children di keduanya.
- Data uji dibersihkan: `wh_movements`/`wh_batches`/`wh_import_rows`/`wh_import_batches` dihapus, item/kategori/gudang dihapus via API (`DELETE` 200 semua) → `categories|items|warehouses|batches` total 0 dan `summary` 0/0/0/0; dua admin uji di-soft-delete. Backend tetap jalan (`/healthz` 200), dev server dihentikan.

