---
title: "WO-04: UI Kelola Resident (list, detail, create, edit, status, dokumen)"
description: "Halaman resident: tabel dengan pencarian/filter/paginasi, halaman detail (status history + dokumen), form create/edit berbasis mirror fields.yaml, aksi ubah status append-only, dan tambah dokumen link Drive."
status: progress
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - ui
  - resident
---

# WO-04: UI Kelola Resident

## Referensi
- SPEC: A-5, A-11, A-12, A-13, AC-3, AC-4, AC-6.
- Routes: `GET/POST /api/admin/residents`, `GET/PUT/PATCH/DELETE /api/admin/residents/:id`,
  `GET/POST /api/admin/residents/:id/status`,
  `GET/POST /api/admin/residents/:id/documents`,
  `GET /api/admin/mothers`.
- Payload: `CreateResidentInput` / `UpdateResidentInput` (partial, `nil` = unchanged),
  `StatusInput{status_code, effective_at, note}`, `DocumentInput{doc_type, url, label}`.
- Aggregate: `ResidentFull{resident, mother, birth, admissions[], current_status, documents[], branch}`.
  **Catatan:** kunci status agregat = `current_status` (bukan `status`).
- Enum: `plugins/resident/schema/fields.yaml`.

## Langkah

### T-1 — Tipe & mirror schema
1. `src/features/resident/types.ts`: `Resident`, `ResidentFull`, `Mother`,
   `BirthRecord`, `Admission`, `CurrentStatus`, `ResidentDocument`, `BranchRef`,
   input types. Semua sesuai swagger.
2. `src/features/resident/fields.ts`: mirror `fields.yaml` (enums gender, religion,
   blood_type, delivery_method, attendant_type, status, document_type) + label
   Indonesia. Header komentar: **"mirror dari plugins/resident/schema/fields.yaml —
   update bersamaan"** (mitigasi A-12).

### T-2 — List (`/residents`)
- Kolom: `resident_code` (mono), `full_name`, gender, `birth_date` (umur turunan),
  `current_status` (Badge), `branch.label ?? branch.key`, aksi.
- Filter: `q` (debounce 300ms), `branch_key` (dari `GET /api/admin/facilities`
  atau daftar cabang — pakai sumber yang tersedia), `status` (WO-01 B-2).
- Paginasi: page size 25, prev/next berbasis `items.length`; tampilkan `total`
  bila tersedia (A-11).
- State: loading skeleton, empty state ("Belum ada data resident"), error alert
  dengan tombol "Coba lagi".
- Tombol "Tambah Resident" hanya bila `hasPermission('resident.write')`.

### T-3 — Detail (`/residents/:id`)
- Kartu ringkas: kode (read-only), nama, gender, tanggal lahir, cabang.
- Kartu status: status terkini + tombol "Ubah Status".
- Tabel riwayat status (append-only, urut terbaru) + catatan.
- Tabel dokumen: `doc_type` (badge), label, URL Drive (tautan `target="_blank"`,
  `rel="noreferrer"`), tombol "Tambah Dokumen".
- Kartu data ibu, kelahiran, admisi (jika ada).
- Tombol Edit (`resident.write`) & Hapus (`resident.delete`) dengan modal konfirmasi.
- 404 → tampilan "Resident tidak ditemukan" + tautan kembali.

### T-4 — Form create/edit
- Section: Data Bayi (wajib: `full_name`, `gender`, `birth_date`, `branch_key`),
  Data Ibu (opsional), Kelahiran (opsional), Admisi (opsional), Status awal
  (create saja), Dokumen (create: opsional).
- `resident_code`: kosongkan = server generate (WO-10 lama); tampilkan hint
  "Kosongkan agar sistem membuat kode otomatis"; tidak bisa diedit setelah dibuat.
- Validasi client: wajib isi, format tanggal `YYYY-MM-DD` (input `type="date"`),
  NIK 16 digit bila diisi, enum dari `fields.ts`.
- Edit memakai `PATCH` **hanya field yang berubah** (diff vs nilai awal); field
  yang dikosongkan dikirim `""` hanya bila diizinkan backend — verifikasi, kalau
  tidak, nonaktifkan pengosongan.
- Setelah sukses: toast + kembali ke detail/list (refetch).

### T-5 — Ubah status & tambah dokumen
- Modal Ubah Status: `status_code` (select enum), `effective_at` (default sekarang),
  `note` (wajib untuk `TRANSFER` — isi tujuan, mis. "JCH"; lihat D-6).
- Modal Tambah Dokumen: `doc_type`, `url` (wajib validasi `https://`), `label`.
- Refetch detail setelah sukses.

## Kriteria Penerimaan
- [AC-1] List menampilkan data nyata dari API; pencarian `q` dan filter
  `branch_key`/`status` bekerja; paginasi prev/next konsisten.
- [AC-2] Create berhasil dengan field wajib minimum; kode otomatis tampil
  (mis. `BBH-26-P-00x`); kode duplikat → 409 ditampilkan ramah.
- [AC-3] Detail menampilkan `current_status`, riwayat status, dan dokumen.
- [AC-4] Ubah status menambah baris riwayat (append-only) tanpa menghapus yang lama.
- [AC-5] Tambah dokumen menampilkan link Drive yang bisa dibuka.
- [AC-6] PATCH mengirim hanya field berubah; respons 400 (enum/format) tampil
  sebagai pesan per-field bila memungkinkan.
- [AC-7] Viewer tanpa `resident.write`/`resident.delete` tidak melihat tombol
  mutasi; 403 server tetap ditangani.
- [AC-8] Hapus → konfirmasi → sukses → list ter-refresh.
- [AC-9] `npm run build` lulus; tidak ada error TS.

## Bukti
Alur create → detail → ubah status ×2 → tambah dokumen → edit → hapus (screenshot
+ catatan respons API). Uji viewer (403/403) dan error validasi.

## Risiko
- Drift `fields.ts` vs `fields.yaml` (A-12) → wajib komentar + checklist review.
- `UpdateResidentInput` tidak mendukung ubah `resident_code` (memang read-only).

## Rencana Pemecahan (arahan Manager 2026-10-28)
WO ini dilaporkan **per lapis**, bukan satu laporan raksasa:

| Lapis | Isi | Status |
|---|---|---|
| **A** | List: `q` debounce 300ms, `branch_key`, `status`, paginasi dari `total`, skeleton/empty/error+retry | **selesai** |
| **B** | Detail: `current_status`, riwayat status, dokumen Drive, ibu/kelahiran/admisi, hapus + edit via modal | belum |
| **C** | Form create/edit + mutasi (status & dokumen), `resident_code` auto, PATCH diff-only, guard `hasPermission` | belum |

## Laporan Lapis A (2026-10-28)

### File
| Berkas | Isi |
|---|---|
| `src/lib/fields.ts` | **Mirror `fields.yaml`**: 8 enum / 40 nilai, label Indonesia apa adanya. Header komentar memuat **tanggal sinkron (2026-10-28)**, versi schema, jumlah enum/nilai, dan perintah `grep` untuk mendeteksi drift. |
| `src/lib/apiError.ts` | Penerjemah error → pesan Indonesia (network/400/401/403/404/409/422/5xx). Pesan mentah backend tidak pernah ditampilkan apa adanya. |
| `src/lib/format.ts` | `formatDate`/`formatDateTime`/`orDash`. **Format tanggal memakai `timeZone: 'UTC'`** — kolom `date` tersimpan tengah malam UTC; tanpa itu tanggal bisa bergeser 1 hari di WIB (UTC+8). |
| `src/api/residents.ts` | Tipe `ResidentListItem`/`ListResidentsResponse` + `listResidents()` (page size 25) + `listBranches()`. |
| `src/hooks/useDebouncedValue.ts` | Debounce generik (dipakai 300ms). |
| `src/pages/admin/residents/ResidentListPage.tsx` | Halaman list. |
| `src/App.tsx` | Rute `/residents` → `ResidentListPage` (placeholder WO-04 dilepas). |

### Catatan implementasi
1. **Filter cabang memakai `GET /api/admin/endpoints`**, bukan `facilities`.
   Alasan: identitas cabang = `endpoints.key` (D-11) — itulah nilai `branch_key`
   yang difilter backend. `facilities` adalah tempat persalinan, bukan cabang.
   Deviasi tercatat di sini. Bila aktor tidak punya `endpoint.read`, `listBranches()`
   mengembalikan `[]` dan dropdown jatuh ke "Semua cabang" (tanpa error).
2. **Perubahan filter mengatur ulang `offset` ke 0** — kalau tidak, offset lama
   bisa berada di luar hasil filter baru (tabel kosong tanpa penjelasan).
3. Saat `state === 'error'`, pesan muncul **dua kali** (Alert di atas + baris
   tabel): disengaja agar pengguna melihat sebab dan tombol **Coba lagi**
   tepat di tempat data seharusnya muncul.
4. `resident_code` kosong → `orDash()` mengembalikan `—` (field memang `omitempty`
   di backend, tidak dikirim saat NULL).

### Gate (browser, `http://100.125.175.34:5173/app/residents`, data nyata)
Data uji: 3 resident (status DIRAWAT/DIKEMBALIKAN/TRANSFER, 2 punya ibu) + 26
resident paginasi; **semua dihapus setelah uji** (DB kembali `{"items":[],"total":0}`).

| # | Uji | Hasil |
|---|---|---|
| 1 | Muat halaman | 3 baris; badge status `Dirawat`/`Dikembalikan`/`Transfer` berwarna; cabang `local`; tanggal `10 Feb 2026` (tanpa pergeseran zona) |
| 2 | Pencarian `Dua` | 150ms → masih 3 baris; setelah debounce → **1 baris** (`Uji WO04 Bayi Dua`), catatan `1 data cocok` → debounce 300ms terbukti |
| 3 | Filter status | `DIKEMBALIKAN` → 1 baris; request `…&status=DIKEMBALIKAN`. Reset → 3 baris (`…&limit=25&offset=0`) |
| 4 | Filter kosong (tanpa `q`/`status`) | request tidak mengirim parameter sia-sia |
| 5 | Paginasi (29 data) | Halaman 1 → 25 baris, `Halaman 1 dari 2 · total 29 data`, tombol Sebelumnya disabled; Halaman 2 → 4 baris, tombol Berikutnya disabled |
| 6 | Empty state (filter `zzz-tidak-ada`) | `Tidak ada resident yang cocok dengan filter…` (pesan kontekstual, bukan tabel kosong), catatan `0 data cocok` |
| 7 | Error state (request di-abort) | Alert `Gagal memuat daftar — Tidak dapat menghubungi server…` + baris tabel dengan tombol **Coba lagi** |
| 8 | Pemulihan | Setelah request normal kembali → 29 data, Alert hilang |
| 9 | `npm run build` | hijau (`tsc -b && vite build`, 120 modul) |
| 10 | Stateless | `grep -rEo "http://(100\\.125\\.175\\.34\\|localhost):(8082\\|5173)" dist` → kosong |

### Catatan pengujian
- Uji dilakukan pada dua tab: tab pertama **tidak terlihat** (`visibilityState:
  hidden`) sehingga screenshot-nya kosong; tab baru dibuka agar terlihat dan
  screenshot diambil dari sana. Ini keterbatasan alat, bukan aplikasi.
- `page.fill('#q', '')` **tidak mengubah state React** pada input controlled di
  tab tersembunyi; pembersihan harus lewat keyboard (`Control+A`, `Backspace`).
  Sudah diperhitungkan agar tidak salah menyimpulkan "filter rusak".

### Betul-betul belum dikerjakan (Lapis B/C)
Detail, form create/edit, ubah status, tambah dokumen, hapus, dan seluruh
tombol mutasi — sesuai arahan pemecahan bertahap.
