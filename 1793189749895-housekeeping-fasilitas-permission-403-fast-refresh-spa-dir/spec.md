---
title: "SPEC: Housekeeping — fasilitas, permission 403, Fast Refresh, SPA_DIR"
description: "Menutup sisa item H-2, H-8, H-9, H-10 dan GAP-14: hapus/relokasi fasilitas, branch_key pada update fasilitas, uji jalur 403 anti-escalation, pemisahan useAuth, dan override spaDir lewat env."
status: draft
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - housekeeping
  - resident
  - facilities
  - rbac
  - spa
---

# SPEC: Housekeeping (fasilitas, 403, Fast Refresh, SPA_DIR)

## Ringkasan
Lima utang teknis kecil dari topik sebelumnya, semuanya **lokal & tidak berisiko besar**, tetapi
masing-masing menutup lubang nyata:

| Item | Inti |
|---|---|
| **H-9** | Fasilitas tidak punya endpoint `DELETE` → data uji harus dibersihkan via SQL |
| **H-10** | `PUT /api/admin/facilities/:id` tidak menerima `branch_key` → field cabang disabled di UI |
| **H-8** | Jalur **403 anti-escalation** belum teruji end-to-end (aktor uji root memegang `*`) |
| **H-2** | `AuthContext.tsx` mengekspor provider + hook → peringatan Fast Refresh |
| **GAP-14** | `spaDir` hard-code relatif CWD → SPA 404 senyap bila server dijalankan dari direktori lain |

## Kondisi Awal (terverifikasi)
- `plugins/resident/plugin.go` — grup `facilities` hanya `GET`, `POST`, `PUT`. **Tidak ada `DELETE`.**
- `plugins/resident/services/query.go:725` — `UpdateFacility(id string, name, facilityType *string,
  isActive *bool)` — **tanpa** parameter `branch_key`. Handler `PUT` karenanya tidak bisa mengubah cabang.
- `facilities` punya `facility_aliases` dengan `ON DELETE CASCADE` ke `facilities`; `birth_records.facility_id`
  memakai **`ON DELETE SET NULL`** → menghapus fasilitas **tidak** memutus riwayat kelahiran.
- `middleware.RequirePermission` (`plugins/auth/middleware`) → `403` bila permission tidak ada.
  `EnsureCanGrant` menolak pemberian grup melebihi hak aktor (`403`) dan menolak perubahan keanggotaan
  grup sendiri bila aktor bukan `*`. **Belum ada uji e2e** karena hanya ada admin root.
- `sub_app/webapp/src/auth/AuthContext.tsx` mengekspor `AuthProvider` **dan** `useAuth()` → Vite
  Fast Refresh melakukan full reload saat file diedit.
- `plugins/pages/plugin.go:20` — `const spaDir = "./sub_app/webapp/dist"`, dibaca langsung di
  `RegisterRoutes` (`p.spaDir = spaDir`); tidak ada env override.

## Keputusan Desain
| # | Keputusan | Nilai |
|---|---|---|
| A-1 | Hapus fasilitas (H-9) | `DELETE /api/admin/facilities/:id`, guard **`resident.delete`** (konsisten dengan `DELETE /residents/:id`). **Hard delete**; alias ikut terhapus (CASCADE); `birth_records.facility_id` → `NULL` (SET NULL) — didokumentasikan sebagai konsekuensi, bukan bug. |
| A-2 | Konflik referensi | Karena DB memakai `SET NULL`, **tidak ada** `409` referensi; namun respons menyertakan jumlah `birth_records` yang terdampak agar pemanggil sadar. Alternatif (tolak bila dipakai) **ditolak** — riwayat tidak boleh memblokir pembersihan data master. |
| A-3 | `branch_key` update (H-10) | Tambah parameter `branchKey *string` ke `UpdateFacility`; validasi via `resolveBranchKey`; `""` (blank) = **cabang dikosongkan/NULL**, `nil` = tidak diubah. |
| A-4 | Uji 403 (H-8) | Buat admin uji **non-root** (grup `viewer`/`admin`) via CLI/API di lingkungan uji, lalu buktikan `403` pada: (a) memberi grup root ke orang lain, (b) mengubah keanggotaan grup sendiri, (c) memanggil endpoint `resident.write` tanpa hak. **Tidak** menambah kode fitur baru — ini verifikasi + test otomatis bila memungkinkan. |
| A-5 | Fast Refresh (H-2) | Pisahkan `useAuth` ke `src/auth/useAuth.ts` (context object dari file ketiga `authContext.ts`) agar `AuthContext.tsx` hanya mengekspor komponen. **Tidak** mengubah perilaku auth. |
| A-6 | `spaDir` env (GAP-14) | `plugins/pages` membaca `SPA_DIR` bila diset, fallback ke `./sub_app/webapp/dist`. Bila direktori/index tidak ada saat registrasi → **log error jelas** (bukan gagal senyap). Perubahan minimal di `internal/`? **Tidak** — `plugins/pages` adalah plugin, aman. |
| A-7 | Bahasa & UI | UI tetap Bahasa Indonesia; identifier kode Inggris. |
| A-8 | Kompatibilitas | Semua perubahan **aditif** (route baru, parameter opsional, env opsional) → tidak memecah klien/konfigurasi existing. |
| A-9 | Kontrak | Swagger + Bruno diperbarui untuk route fasilitas. |

## Cakupan Fungsional
- Menghapus fasilitas dari UI & API; pesan jelas bila gagal.
- Mengubah cabang fasilitas lewat `PUT` (field tidak lagi disabled di UI).
- Bukti tertulis jalur 403 untuk tiga skenario.
- Fast Refresh bersih (edit `AuthContext.tsx` tidak memicu full reload).
- Server bisa dijalankan dari direktori lain dengan `SPA_DIR` absolut.

## Non-Goals
- Tidak menambah soft-delete / audit trail fasilitas.
- Tidak menambah permission baru (`facility.*`); pakai `resident.*`.
- Tidak mengubah aturan anti-lockout/anti-escalation (hanya menguji).
- Tidak menyentuh image produksi (topik `1793189749848`).
- Tidak melakukan upgrade dependensi (topik yang sama).

## Gap yang Diketahui
| GAP | Deskripsi | Rencana |
|---|---|---|
| GAP-1 | `facilities` tanpa `DELETE` (H-9) | **WO-01** |
| GAP-2 | `UpdateFacility` tanpa `branch_key` (H-10) | **WO-01** |
| GAP-3 | Fast Refresh terganggu ekspor campuran (H-2) | **WO-02** |
| GAP-4 | `spaDir` tak bisa di-override (GAP-14) | **WO-02** |
| GAP-5 | Jalur 403 belum teruji e2e (H-8) | **WO-03** |

## Kriteria Penerimaan (SPEC-level)
- **AC-1** — `DELETE /api/admin/facilities/:id` → `204`; id tak ada → `404`; tanpa token → `401`;
  tanpa `resident.delete` → `403`.
- **AC-2** — Setelah hapus, `facility_aliases` terkait hilang dan `birth_records.facility_id`
  yang menunjuk ke fasilitas itu menjadi `NULL` (dibuktikan dengan query).
- **AC-3** — `PUT /api/admin/facilities/:id` menerima `branch_key`; blank → `NULL`; nilai tak dikenal
  → `400`; `nil` (tidak dikirim) → tidak berubah.
- **AC-4** — UI: tombol Hapus fasilitas (permission-aware, konfirmasi) dan field Cabang aktif di
  mode edit.
- **AC-5** — Tiga skenario 403 (A-4) terbukti dengan output nyata; hasil dicatat di WO-03.
- **AC-6** — `AuthContext.tsx` hanya mengekspor komponen; `useAuth` di file terpisah; `npm run build`
  lulus; edit file AuthContext tidak memicu full reload (diverifikasi manual).
- **AC-7** — `SPA_DIR` menimpa `spaDir`; tanpa env → perilaku lama identik; direktori hilang →
  log error eksplisit saat registrasi.
- **AC-8** — Swagger (`paths` bertambah untuk `DELETE /facilities/{id}`) + Bruno request baru.
- **AC-9** — `go build ./...` + `go test ./plugins/...` lulus; `internal/` tidak berubah.
- **AC-10** — Tanpa push.

## Risiko
- **Hapus fasilitas** mem-NULL-kan referensi riwayat → dimitigasi A-1/A-2 + informasi dampak.
- **Uji 403** bisa mengunci diri sendiri bila salah konfigurasi → gunakan admin uji terpisah,
  jangan menurunkan hak root terakhir (anti-lockout sudah ada).
- **`SPA_DIR`** salah nilai → SPA 404; dimitigasi log error eksplisit (A-6).
