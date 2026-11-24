---
title: wo 01 clean data
description: Reset data semua segment via console command, jaga admin/RBAC
status: complete
created: "2026-11-24"
updated: "2026-11-24"
---
# wo 01 clean data

## Ringkasan
Buat **console command** Go untuk reset data semua segment (resident, hsh,
sec, visit, warehouse) TANPA menghapus user/admin & RBAC. Jalankan di DB dev
untuk membersihkan data lama.

## Detail
- Console CLI ada di `internal/console` + `cli.go` tiap plugin — ikuti pola
  command yang sudah ada (mis. di plugin), command baru diletakkan di plugin
  terkait atau satu command lintas-segment.
- Urutan DELETE/TRUNCATE aman-FK (dari relasi yang sudah dicek):
  1. birth_records, admissions, resident_status_history, resident_documents,
     residents, mothers, facility_aliases, facilities
  2. hsh_entries, hsh_item_aliases, hsh_items, hsh_categories, hsh_periods
  3. sec_entries, sec_item_aliases, sec_items, sec_categories, sec_periods
  4. visit_visitor_merges, visit_status_history, visit_visits, visit_visitors
  5. wh_movements, wh_batches, wh_items, wh_warehouses, wh_categories
- Import tables (hsh_import_*, dll) ikut dibersihkan (ro&batch).
- HANYA data fitur — JANGAN sentuh: admins, admin_groups, groups,
  group_permissions, permissions, migrations, migration_targets, endpoints,
  endpoint_audit, plugin registry, ingest_*.
- Format command bebas mengikuti konvensi repo, mis. `console db clean-data`
  (dokumentasikan di help text).

## Tugas
1. [x] Implementasi console command reset data lintas segment (urutan FK aman).
2. [x] Jalankan di container & verifikasi: counts = 0 untuk semua tabel di atas,
       admins tetap 20 + grup RBAC utuh.
3. [x] Health: login tetap berfungsi (tes 1 login admin).
4. [x] Update WO + konteks dengan hasil.

## Acceptance
- Semua tabel data fitur kosong (0 baris).
- Admin/RBAC/framework utuh.
- Login admin masih bekerja.

### Bukti
- Kode: `132fdb0` (command), `6144915` (`--reseed`). Tanpa push.
- Dry run sebelum reset: 8.650 baris di 37 tabel.
- Eksekusi 1: 8.650 baris dihapus → semua tabel fitur 0.
- Dijaga: `admins` 20, `members` 3, `groups` 6, `group_permissions` 39,
  `admin_groups` 16, `permissions` 32, `endpoints` 1, `migrations` 33,
  `ingest_sessions` 355, `ingest_messages` 605 — tidak berubah.
- Login: `POST /api/admin/login` (root@admin.local) → token 216 char;
  `GET /api/admin/residents` 200; `GET /api/admin/dashboard/summary` 200;
  `GET /healthz` 200.
- Test unit: `go test ./plugins/maintenance/...` hijau (kunci tabel terlarang,
  dedup, urutan FK child-before-parent, paritas seed vs migrasi, bentuk UUIDv7).

## Catatan
- Konfirmasi user: clean via console command (boleh).
- **Perlu tindakan lanjutan (ditemukan saat eksekusi):** reset ikut menghapus
  seed kategori `hsh` (7) & `sec` (2). Importer kedua plugin **tidak**
  auto-create kategori → semua baris akan ditolak bila tidak dipulihkan.
  Ditambahkan flag `--reseed` yang mengembalikan seed tersebut (idempotent,
  `ON CONFLICT (code) DO NOTHING`). Tanpa `--reseed` perintah mencetak peringatan.
- Fasilitas & gudang **sengaja tidak** di-seed ulang: keduanya master data
  operasional milik operator, bukan enum kode.

## Catatan Implementasi (Dev)

- `plugins/maintenance/plugin.go:1-20` — plugin baru `maintenance` (ID
  `maintenance`) dengan banner alasan: fitur wajib plugin (aturan repo), dan
  operasi destruktif tidak boleh punya permukaan HTTP.
- `plugins/maintenance/plugin.go:49` — `RegisterRoutes` sengaja no-op.
  Kenapa: tanpa route, tidak ada jalur network (bahkan dengan RBAC) yang bisa
  memicu reset data.
- `plugins/maintenance/plugin.go:55-57` — `ConsoleCommands` mengembalikan group
  `maintenance`, mengikuti pola `wh`/`visit`/`dashboard` (`cli.go` per plugin).
- `plugins/maintenance/services/service.go:42-92` — `CleanupTables` = 37 tabel
  dalam urutan children→parents. Kenapa urutan penting: FK `hsh_items.category_id`,
  `visit_visits.visitor_id`, `wh_batches.item_id` dideklarasikan **tanpa**
  `ON DELETE CASCADE`, jadi DELETE induk lebih dulu akan gagal.
- `plugins/maintenance/services/service.go:94-102` — `PreservedTables`
  didokumentasikan di kode (admins/RBAC/framework/ingest) supaya daftar
  "jangan disentuh" bukan hanya catatan WO.
- `plugins/maintenance/services/service.go:135-157` — `Counts()` memakai
  `Migrator().HasTable` dan menandai tabel hilang sebagai `Missing` bukan error.
  Kenapa: perintah harus tetap jalan sebelum semua plugin pernah dimigrasi.
- `plugins/maintenance/services/service.go:159-196` — `Clean()` satu transaksi
  (BEGIN…COMMIT) dengan rollback + recover. Kenapa: kegagalan FK di tengah
  tidak boleh meninggalkan DB separuh terhapus.
- `plugins/maintenance/services/service.go:198-200` — `quoteIdent` (double quote)
  hanya untuk nama tabel konstanta, bukan input user.
- `plugins/maintenance/cli.go:52-54` — GORM di-`Session` dengan
  `logger.Silent`. Kenapa: `internal/db` memakai `logger.Info`, dan log SQL
  menyela tabel laporan sehingga output tidak terbaca.
- `plugins/maintenance/cli.go:60-72` — default **dry run**; `--yes` wajib untuk
  eksekusi. Kenapa: pencegahan salah hapus pada perintah destruktif.
- `plugins/maintenance/cli.go:96-110` — flag `--reseed` memanggil `Reseed`
  setelah `Clean` sukses, mencetak peringatan bila tidak dipakai.
- `plugins/maintenance/services/service.go:218-236` — `HSHCategorySeeds` /
  `SecCategorySeeds` menyalin blok seed migrasi. Kenapa: importer resolve
  kategori lewat tabel `*_categories` dan MENOLAK baris untuk kode tak dikenal,
  jadi wipe kategori = impor mustahil.
- `plugins/maintenance/services/service.go:241-275` — `Reseed` idempotent
  (`ON CONFLICT (code) DO NOTHING`) dalam satu transaksi; `is_active` dipaksa
  TRUE karena seed migrasi juga TRUE.
- `plugins/maintenance/services/seed_test.go:12-80` — test paritas membaca file
  migrasi `.up.sql` dan membandingkan daftar kode kategori dua arah. Kenapa:
  mencegah seed Go basi ketika migrasi menambah kategori baru.
- `plugins/maintenance/services/service_test.go:9-40` — test memastikan tidak
  ada tabel terjaga (termasuk semua `ingest_*`) yang masuk daftar cleanup.
- `plugins/maintenance/services/service_test.go:56-110` — test urutan
  child-before-parent untuk 26 pasangan FK nyata. Kenapa: urutan bersifat
  load-bearing, mudah rusak saat menambah tabel.
- `cmd/console/main.go:25` — plugin didaftarkan HANYA di console (bukan server).
  Kenapa: tanpa route, mendaftarkannya di server hanya menambah dependensi DB
  tanpa manfaat.
- `plugins/maintenance/cli.go:117-131` — `printCounts` memakai `tabwriter`.
  Kenapa: laporan kolom rapi tanpa campur log SQL, memudahkan verifikasi manual.
