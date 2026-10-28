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

## Progress
| Lapis | Cakupan | Commit | Status |
|---|---|---|---|
| A | List: `q` (debounce 300ms), filter `branch_key`/`status`, paginasi dari `total`, state UI (skeleton/empty/error+retry) | `d05ceb7` | ✅ selesai |
| B | Detail `/residents/:id`: `current_status`, riwayat status, dokumen Drive, ibu/kelahiran/admisi, hapus + konfirmasi, 404 | `00a918b` | ✅ selesai |
| C | Form create/edit (`resident_code` auto + read-only), modal Ubah Status, modal Tambah Dokumen, PATCH diff-only, guard `resident.write` | `a6406ca` | ✅ selesai |

Catatan Lapis B:
- Tombol **Edit sengaja belum dirender** (form belum ada) — tidak memasang tombol mati. Akan muncul di Lapis C.
- Modal hapus dimount hanya bila `canDelete` → viewer tidak punya jalur ke UI mutasi.
- Detail memakai `current_status`, **bukan** `status`.
- Insiden: `git add -A` sempat dijalankan dari dalam `.historic` (bukan repo) → commit kode ikut memuat `bin/server`+`bin/console` (±80 MB). Sudah di-reset (mixed) + commit ulang bersih. Mitigasi: H-5 (`bin/` belum di-gitignore).

Catatan Lapis C (koreksi asumsi Manager):
- **Pengosongan field DIDUKUNG** backend: `UpdateResident` memakai `nilIfBlank` untuk `nik`, `bpjs_no`, `birth_place`, `religion_code`, `blood_type_code`, `mother_id` → `""` menjadi NULL. Instruksi awal ("nonaktifkan pengosongan") **salah** dan tidak dijalankan — menonaktifkan akan menghilangkan fungsi yang sudah bekerja.
- `resident_code` kosong saat create → kunci **tidak dikirim** sama sekali (bukan `""`), server yang generate.
- `full_name` kosong → 400 (ditolak server).
- `mother`/`birth`/`admission` tidak ada di `UpdateResidentInput` → form edit tidak menampilkannya (diberi Alert). Tercatat sebagai **H-6**.
- Dokumen belum bisa dihapus/diedit — API hanya `GET`/`POST` (H-7).
- Guard: `hasPermission('resident.write')` dipakai di `ResidentFormPage`, `ResidentDetailPage`, `ResidentListPage`; `/residents/new` untuk viewer → "Tidak berwenang"; API 403 untuk create/PATCH/status/dokumen.

## Rencana Pemecahan (arahan Manager 2026-10-28)
WO ini dilaporkan **per lapis**, bukan satu laporan raksasa:

| Lapis | Isi | Status |
|---|---|---|
| **A** | List: `q` debounce 300ms, `branch_key`, `status`, paginasi dari `total`, skeleton/empty/error+retry | **selesai** |
| **B** | Detail: `current_status`, riwayat status, dokumen Drive, ibu/kelahiran/admisi, hapus via modal | **selesai** |
| **C** | Form create/edit + mutasi (status & dokumen), `resident_code` auto, PATCH diff-only, guard `hasPermission` | **selesai** |

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

## Laporan Lapis B (2026-10-28)

Lingkup: **halaman detail** `/residents/:id` + jalur navigasi list → detail +
aksi hapus dengan modal konfirmasi. Form create/edit, ubah status, dan tambah
dokumen tetap di Lapis C.

### File
| Berkas | Isi |
|---|---|
| `src/pages/admin/residents/ResidentDetailPage.tsx` (baru) | Halaman detail: `current_status`, riwayat status, dokumen, ibu, kelahiran, admisi, hapus |
| `src/pages/admin/residents/DetailRow.tsx` (baru) | `DetailRow` / `DetailText` — baris `dt/dd` seragam untuk semua kartu |
| `src/api/residents.ts` (ubah) | Tipe agregat (`ResidentFull`, `ResidentDocument`, `StatusHistoryRow`, …) + `getResident`, `listResidentStatus`, `listResidentDocuments`, `deleteResident` |
| `src/App.tsx` (ubah) | Route `/residents/:id` di dalam grup `RequireAuth` + `AdminLayout` |
| `src/pages/admin/residents/ResidentListPage.tsx` (ubah) | Nama resident jadi tautan ke detail; kolom cabang memakai label endpoint (fallback key) |

Tidak ada perubahan di `internal/`, `plugins/`, atau build backend.

### Catatan implementasi
1. **`current_status`, bukan `status`** — kartu “Status terkini” membaca
   `full.current_status?.status_code` (hasil LATERAL subquery WO-01) dan
   menampilkan `effective_at` + `note`. Tidak ada field `status` di UI.
2. **Riwayat append-only** — sumbernya `GET /residents/:id/status` yang sudah
   terurut terbaru→terlama dari server; UI tidak mengurutkan ulang dan tidak
   menawarkan hapus baris riwayat.
3. **Dokumen Drive** — `target="_blank"` + `rel="noreferrer"` (wajib, diverifikasi
   lewat DOM), `doc_type` memakai `enumLabel('document_type', …)`.
4. **Edit tidak dipasang di Lapis B** — sengaja, karena form-nya baru ada di
   Lapis C. Tombol “Edit” yang belum berfungsi tidak dirender supaya tidak ada
   tombol mati. Tombol **Hapus** hanya muncul bila `hasPermission('resident.delete')`
   dan modal konfirmasinya pun tidak dipasang (`{canDelete && <Modal …>}`).
5. **Viewer** — tanpa `resident.write` halaman menampilkan Alert “Anda masuk
   sebagai pengamat…”; tanpa `resident.delete` tidak ada tombol Hapus. 403 dari
   server tetap ditangani (`deleteError` → Alert di dalam modal), jadi bukan
   sekadar menyembunyikan tombol.
6. **Umur manusiawi** — `ageLabel()` menampilkan `11 bulan` untuk < 2 tahun dan
   `N tahun` selebihnya (bukan `0 tahun`).
7. **Baris kosong tidak dipaksakan** — “Fasilitas (ID)” disembunyikan bila
   `facility_id` null; seksi ibu/kelahiran/admisi punya pesan kosong masing-masing.
8. **Label cabang** — detail memakai `branch.label ?? branch.key ?? branch_key`;
   list memakai peta dari `listBranches()` dengan fallback key mentah.

### Gate (browser, `http://100.125.175.34:5173/app/residents/:id`, data nyata)
Data uji: 1 resident lengkap (ibu, kelahiran, admisi, 3 baris riwayat status,
2 dokumen) + 1 admin viewer sementara. **Semua dihapus setelah uji** — DB kembali
`{"items":[],"total":0}` dan admin `Viewer Uji` dihapus.

| # | Uji | Hasil |
|---|---|---|
| 1 | Klik nama di list | Pindah ke `/app/residents/<id>`; judul tab “Detail Resident · BBH Data Board — Admin” |
| 2 | Kartu data bayi | `20 Nov 2025 · 11 bulan`, agama `Hindu`, gol darah `O positif`, cabang `Instance ini` |
| 3 | Status terkini | `Transfer`, “Berlaku sejak 01 Feb 2027”, catatan “Transfer ke JCH” |
| 4 | Riwayat status | 3 baris terbaru→terlama (`01 Feb 2027`, `10 Jan 2027`, `25 Nov 2025`), catatan kosong → `—` |
| 5 | Dokumen | 2 entri, badge `Folder Google Drive` + `Media`; DOM: `target="_blank"`, `rel="noreferrer"` untuk keduanya |
| 6 | Ibu / Kelahiran / Admisi | Terisi lengkap (`2900 g`, `48 cm`, `Anak ke- 2`, `Normal`, `Bidan`); “Fasilitas (ID)” tidak muncul karena null |
| 7 | ID tidak ada | `Resident tidak ditemukan` + tombol “Kembali ke daftar” (bukan halaman error generik) |
| 8 | Hapus (root) | Modal “Hapus resident?” → “Ya, hapus” → toast `Resident dihapus.` → redirect `/residents`, `0 data cocok` |
| 9 | Viewer: GET detail | 200; tombol Hapus **tidak dirender**, Alert pengamat tampil |
| 10 | Viewer: DELETE API | `403` `{"error":"izin ditolak: butuh resident.delete"}` |
| 11 | `npm run build` | hijau (`tsc -b && vite build`, 122 modul) |
| 12 | Stateless | `grep -rEo "http://(100\.125\.175\.34\|localhost):(8082\|5173)" dist` → kosong |

### Catatan pengujian
- Seed lewat API sempat gagal `400` karena dua nilai enum salah tebak
  (`status=PULANG`, `doc_type=akta_lahir`); enum sebenarnya hanya
  `DIRAWAT/DIKEMBALIKAN/TRANSFER/MENINGGAL` dan
  `drive_folder/media/other` (lihat `fields.yaml`). Ini justru mengonfirmasi
  validasi server bekerja — bukan bug UI.
- Saat file diubah bertahap, Vite sempat gagal reload (`500`) karena sintaks
  setengah jadi; build final bersih. Tidak ada jejak di commit.

### Betul-betul belum dikerjakan (Lapis C)
Form create/edit (`/residents/baru`, `/residents/:id/edit`), modal Ubah Status,
modal Tambah Dokumen, PATCH diff-only, dan guard `hasPermission('resident.write')`
untuk tombol mutasi. Tombol “Edit” sengaja belum dirender sampai form-nya ada.

## Laporan Lapis C (2026-10-28)

Lingkup (arahan Manager, “tetap 3 hal”): form create, PATCH diff-only, modal status
+ dokumen. Ditambah guard `resident.write` di setiap tombol mutasi.

### File
| Berkas | Isi |
|---|---|
| `src/pages/admin/residents/ResidentFormPage.tsx` (baru) | Satu komponen untuk create (`/residents/new`) & edit (`/residents/:id/edit`) |
| `src/pages/admin/residents/StatusModal.tsx` (baru) | Modal Ubah Status (append-only, `note` wajib untuk TRANSFER) |
| `src/pages/admin/residents/DocumentModal.tsx` (baru) | Modal Tambah Dokumen (URL wajib `https://`) |
| `src/api/residents.ts` (ubah) | `CreateResidentPayload`, `UpdateResidentPayload`, `createResident`, `patchResident`, `addResidentStatus`, `addResidentDocument` |
| `src/pages/admin/residents/ResidentDetailPage.tsx` (ubah) | Tombol Edit/Ubah status/+Status/+Dokumen (guard `resident.write`) + mount kedua modal |
| `src/pages/admin/residents/ResidentListPage.tsx` (ubah) | Tombol “+ Tambah resident” (guard `resident.write`) |
| `src/App.tsx` (ubah) | Rute `/residents/new` (sebelum `:id`) dan `/residents/:id/edit` |
| `.gitignore` (ubah) | `bin/` diabaikan — temuan **H-5** (lihat catatan) |

### Catatan implementasi
1. **`resident_code` kosong = auto.** Form tidak mengirim `resident_code` bila
   field dibiarkan kosong (kunci tidak ada di payload sama sekali, bukan `""`),
   sehingga server memakai generator WO-10. Hint “Kosongkan agar sistem membuat
   kode otomatis” tampil di bawah field. Di halaman edit field ini `disabled` +
   hint “Kode rekam medis tidak dapat diubah”.
2. **PATCH diff-only terbukti dari payload nyata**, bukan dari kode saja. Edit
   nama + kosongkan BPJS + ubah tempat lahir + ubah agama menghasilkan body:
   `{"full_name":"…","bpjs_no":"","birth_place":"Denpasar","religion_code":"KATOLIK"}`
   — hanya 4 kunci, tidak ada `nik`, `gender`, `birth_date`, dll.
3. **Pengosongan field DIDUKUNG** (koreksi terhadap kekhawatiran awal):
   `services.UpdateResident` memakai `nilIfBlank` untuk `nik`, `bpjs_no`,
   `birth_place`, `religion_code`, `blood_type_code`, jadi `""` → NULL. Diuji:
   `bpjs_no` dikosongkan → tampil `—`. Karena itu pengosongan TIDAK dinonaktifkan.
   Yang memang tidak bisa dikosongkan: `full_name` (server menolak 400).
   `resident_code`, `mother`, `birth`, `admission` tidak ada di
   `UpdateResidentInput` → form edit tidak menampilkannya (Alert menjelaskan).
4. **Tombol Simpan tanpa perubahan** → toast “Tidak ada perubahan untuk disimpan.”
   dan **tidak ada request PATCH sama sekali** (diuji: 0 request).
5. **`note` wajib untuk TRANSFER** ditegakkan di klien (inline error, tanpa
   request) *dan* tidak bergantung pada itu — server tetap sumber kebenaran.
6. **Dokumen**: URL wajib diawali `https://`; `http://…` ditolak inline tanpa
   request. Tautan hasil tersimpan tetap `target="_blank"` + `rel="noreferrer"`.
7. **Guard rute form.** Pengamat yang mengetik `/residents/new` langsung melihat
   “Tidak berwenang”, bukan form yang pasti gagal 403 saat disimpan.
8. Modal hapus/status/dokumen hanya **dimount** bila izinnya ada, jadi viewer
   tidak punya jalan ke UI mutasi mana pun (`dialogs: 0` diuji di DOM).

### Gate (browser, `http://100.125.175.34:5173`, data nyata)
Satu alur penuh sebagai root + satu sesi viewer. Data uji (1 resident lengkap +
1 admin viewer) **dihapus setelah uji**; DB kembali `{"items":[],"total":0}` dan
daftar admin kembali 4 akun semula.

| # | Uji | Hasil |
|---|---|---|
| 1 | Create (semua seksi terisi) | `201`; POST body memuat `mother`/`birth`/`admission`/`status`, **tanpa** `resident_code`; redirect ke detail, kode otomatis `BBH-26-P-001` |
| 2 | Ubah status TRANSFER tanpa catatan | error inline “Status Transfer wajib disertai catatan tujuan.”, **0 request** |
| 3 | Ubah status TRANSFER + catatan | `201`; riwayat 1 → **2 baris**, “Status terkini” ikut berubah |
| 4 | Ubah status lagi (DIRAWAT) | riwayat **3 baris**; baris lama TIDAK tertimpa (append-only terbukti) |
| 5 | Tambah dokumen `http://…` | inline “URL harus diawali https://”, **0 request** |
| 6 | Tambah dokumen `https://…` | tersimpan; DOM: `target="_blank"`, `rel="noreferrer"` |
| 7 | Edit tanpa perubahan | toast info, **0 request PATCH** |
| 8 | Edit 4 field | PATCH body **hanya 4 kunci** (lihat catatan 2); `bpjs_no` jadi `—` |
| 9 | Hapus | modal konfirmasi → redirect list, `0 data cocok` |
| 10 | Viewer: list | tombol “+ Tambah resident” **tidak ada** |
| 11 | Viewer: detail | hanya tombol “← Daftar”; tidak ada Edit/Ubah status/Hapus/+Status/+Dokumen; `dialog` = 0 |
| 12 | Viewer: `/residents/new` | “Tidak berwenang” + kembali ke daftar |
| 13 | Viewer: API | GET detail `200`; POST create, PATCH, POST status, POST dokumen semuanya **`403`** |
| 14 | `npm run build` | hijau (`tsc -b && vite build`, 125 modul) |
| 15 | Stateless | `grep -rEo "http://(100\.125\.175\.34\|localhost):(8082\|5173)" dist` → kosong |

### Temuan H-5 (dikerjakan, bukan WO)
`bin/` tidak ada di `.gitignore` — inilah akar insiden commit 80 MB pada Lapis B
(bukan semata salah direktori). Sudah ditambahkan `/bin/` beserta komentar alasan;
`git check-ignore` mengonfirmasi `bin/server` & `bin/console` terabaikan. Ikut
memperbaiki baris `sub_app/webapp/tsconfig.tsbuildinfo` yang sempat tergabung
dengan komentar di baris berikutnya.

### Sisa / belum dikerjakan
- Data ibu, kelahiran, dan admisi **belum bisa diubah** setelah dibuat —
  `UpdateResidentInput` tidak punya field-nya. Butuh WO backend terpisah bila
  diinginkan; saat ini Alert di form edit menyatakannya terang-terangan.
- Dokumen belum bisa dihapus/diedit (backend belum menyediakan route).
- `resident_code` eksplisit di form create belum diuji dengan kode duplikat →
  409 (jalur ini ada di backend & sudah diuji di WO-04 backend).
