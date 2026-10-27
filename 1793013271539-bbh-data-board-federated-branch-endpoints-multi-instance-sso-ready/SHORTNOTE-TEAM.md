---
title: 'SHORTNOTE TEAM: Federated Branch Endpoints'
description: 'Ringkasan untuk seluruh tim: apa yang dibangun, kondisi kode, jebakan, urutan WO, aturan yang tidak boleh dilanggar, model izin, checklist DoD, dan batasan. Termasuk status: WO-01..WO-04 complete, WO-05 housekeeping approved.'
status: draft
created: "2026-10-27"
updated: "2026-10-27"
---

# SHORTNOTE — Federated Branch Endpoints

**Untuk:** Tim Developer
**Topic:** `1793013271539-bbh-data-board-federated-branch-endpoints-multi-instance-sso-ready`
**Baseline kode:** `bbfeb4a` (setelah WO-01..WO-04)

---

## 1. Apa yang Kita Bangun

Satu aplikasi, banyak instance. **Pusat** dan **cabang** menjalankan kode yang **sama persis**, masing-masing dengan DB sendiri.

```
Admin ──login──► PUSAT           (main token)
        │
        └──pilih portal──► CABANG (portal token)
                              │
        menu business ────────┘  (data cabang)
        List Portal  ──► PUSAT   (selalu)
        Admin Profile ──► dua versi (pusat & cabang)
```

**Prinsip inti: identitas dari pusat, izin dari cabang.**

---

## 2. ⚠️ Dua Git Repo Terpisah (baca ini dulu)

| Repo | Lokasi | Isi |
|---|---|---|
| **Kode** | `.git` di root | source code saja |
| **Historic** | `.historic/.database/.git` | dokumen SPEC/WO/shortnote |

`.gitignore:18` **sengaja** meng-ignore `.historic/` agar dua repo tidak tercampur.

| Kebutuhan | Perintah benar | Perintah salah |
|---|---|---|
| Commit dokumen | `historic git add -A && historic git commit` | `git add .` |
| Verifikasi commit dokumen | `historic git log` | `git log` ← "not a valid object name", **normal** |

> Pelajaran umum: `cmd | tail -N` mengembalikan **exit code `tail`**, bukan `cmd`. Status gagal bisa tertelan. Jangan pakai pola itu kalau status penting.

---

## 3. Kondisi Kode Sekarang

**Plugin yang ada:** `auth`, `endpoint`, `federation`, `pages`, `plugin_registry`, `portalsso`

**Aturan user (wajib):** fitur baru **HARUS** jadi plugin di `plugins/<id>/`. **Jangan** menaruh kode fitur di `internal/` — itu hanya untuk framework/bootstrap.

**Baseline terverifikasi:** `go build ./...` OK · `go vet ./...` OK · `go test ./...` 10 paket lulus · `gofmt -l` bersih

---

## 4. Jebakan Paling Sering

| # | Jebakan | Akibat |
|---|---|---|
| 1 | **`bin/` tidak ikut ter-update** oleh `go build ./...` | Fitur tampak "rusak". **Selalu:** `go build -o bin/console ./cmd/console` |
| 2 | Tabel `migrations`/`migration_targets` **sudah ada** | Migrator wajib `CREATE TABLE IF NOT EXISTS` |
| 3 | `api` group sudah ber-prefix `/api` | Daftarkan path relatif (`/business/*path`), bukan `/api/business` → dulu jadi `/api/api/...` |
| 4 | Partial update menyalin seluruh struct | Field ter-reset diam-diam. **Pakai pointer + `Flags().Changed`** |
| 5 | `os.Getenv` + `godotenv.Overload()` | `.env` **menimpa** env proses. Untuk instance kedua, pakai working dir sendiri |
| 6 | `pkill -f "bin/server"` di dalam `sh -c` | Mematikan shell-nya sendiri. Pakai `pkill -9 -f "bin/serve[r]"` |
| 7 | Error I/O diabaikan pada jalur prompt | Prompt tak terlihat, input tetap jalan |

---

## 5. Urutan WO

| WO | Isi | Status |
|---|---|---|
| **01** | Endpoint registry (`endpoints`, `endpoint_audit`, CLI `endpoint`, API) | ✅ `a13645e` |
| **02** | Captive portal routing (dua token, anti-loop, error klasifikasi) | ✅ `51b4f06` |
| **04** | RBAC (`permissions`/`groups`/`admin_groups`), ganti `is_root` | ✅ `1831ec2` |
| **03** | SSO admin-only (HS256 lokal, enrollment, otorisasi lokal) | ✅ `bbfeb4a` |
| **05** | Housekeeping: swagger, `.env.example`, guard `SSO_SHARED_SECRET` | 🔄 approved |

---

## 6. Tiga Aturan yang Tidak Boleh Dilanggar

### 🔴 1. List Portal & login **selalu** di pusat
Kalau ikut berpindah ke cabang lalu cabang mati, admin **terkunci** tanpa jalan keluar.

### 🔴 2. Token tidak pernah membawa peran
```json
{ "iss": "local", "sub": "0193...", "atr": "a@b.c", "aud": ["cabang-jkt"],
  "typ": "portal", "iat": ..., "exp": ... }
```
Tidak ada `role`, `lvl`, `groups`, `permissions`. **Cabang menghitung izin dari DB-nya sendiri, per request.**
Kalau peran ditulis di token, pusat bisa menyelipkan hak tinggi ke cabang — itu eskalasi.

### 🔴 3. Fail-closed, tapi bawaan ramah
- Tanpa grup sama sekali → **tanpa izin** (403), bukan "diizinkan default".
- Admin baru → otomatis grup **`viewer`** → **bisa melihat, belum bisa mengubah**.

---

## 7. Model Izin

```
admins ──< admin_groups >── groups ──< group_permissions >── permissions
```

| Grup | Lihat | Ubah | Hapus | Atur grup | Audit |
|---|---|---|---|---|---|
| `viewer` ← **default** | ✅ | ❌ | ❌ | ❌ | ❌ |
| `admin` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `root` | ✅ | ✅ | ✅ | ✅ | ✅ |

- Izin efektif = **gabungan** semua grup **aktif**.
- `is_root` **diganti** grup `root` ber-permission `*` — kolomnya sudah **dihapus**.
- `is_default` dijaga **unique partial index** (mustahil dua grup default).
- Anti-lockout: admin `*` terakhir tidak bisa dihapus/dicabut; grup `is_system` tidak bisa dihapus.

---

## 8. Keamanan yang Wajib Ada

| Ancaman | Mitigasi |
|---|---|
| Token dipakai di cabang lain | Klaim `aud` → **403** |
| Menyusup tanpa terdaftar | Harus terdaftar & disetujui di `admins` cabang → **403** |
| Privilege escalation | Pelaku tidak boleh memberi izin melebihi miliknya |
| Loop pusat↔cabang | `X-Forwarded-Instance` hop lanjutan → **409** |
| Kunci bocor di log | Guard tidak pernah mencetak nilai kunci; audit hanya metadata |
| **Kunci dipakai ulang** | `SSO_SHARED_SECRET` **harus** ≠ `APP_KEY` → kalau sama, **start ditolak** |

---

## 9. Env Wajib

| Variabel | Fungsi | Catatan |
|---|---|---|
| `INSTANCE_KEY` | Identitas instance (default `local`) | Dipakai audit & `X-Forwarded-Instance` |
| `SSO_SHARED_SECRET` | HS256 key pusat+cabang | **WAJIB beda dari `APP_KEY`**; `openssl rand -base64 32` |
| `FEDERATION_FALLBACK_LOCAL` | Fallback ke `local` saat endpoint gagal | **Default mati** — fallback senyap berbahaya |

> SSO **nonaktif + log jelas** bila `SSO_SHARED_SECRET` kosong (bukan crash) — cabang yang tidak memakai SSO harus tetap start.

---

## 10. Definition of Done

- [ ] Semua AC lulus dengan **bukti output nyata** — bukan asumsi
- [ ] Yang belum teruji ditulis **"belum terbukti"**, jangan diklaim lulus
- [ ] Migrasi **dipisah** dari penghapusan kolom (bisa pulang)
- [ ] Seed idempotent (`ON CONFLICT DO NOTHING`)
- [ ] Uji **negatif** (pastikan aksi berbahaya **ditolak**)
- [ ] `go build -o bin/console ./cmd/console` eksplisit
- [ ] `go build ./... && go vet ./... && go test ./...` lulus
- [ ] `gofmt -l` bersih
- [ ] Verifikasi commit dokumen dengan `historic git log`
- [ ] **Tidak ada push ke remote** — commit lokal saja

---

## 11. Batasan yang Disepakati

- **Admin saja** — federasi/SSO untuk `member` **ditunda**
- Tidak ada sinkronisasi/agregasi data antar cabang
- Tidak ada DB lintas jaringan — **HTTP saja**
- Pencabutan token instan belum ada → `exp` portal token **pendek (15 menit)**
- RS256/JWKS siap di interface, belum diimplementasi
- UI pengelolaan grup menyusul; fase ini CLI + API

---

## 12. Referensi

| Dokumen | Isi |
|---|---|
| `SPEC.md` | D-1..D-25, AC-1..AC-34 |
| `WO-01-endpoint-registry.md` | endpoint registry |
| `WO-02-captive-portal-routing.md` | routing & dua token |
| `WO-03-sso-admin-only.md` | SSO admin-only |
| `WO-04-rbac-groups-permissions.md` | RBAC |
| `WO-05-housekeeping-docs-env.md` | housekeeping (approved) |
| `work-order.md` | indeks + dependency + status akhir |

---

**Pertanyaan sebelum mulai? Tanyakan sekarang, jangan menebak.**
