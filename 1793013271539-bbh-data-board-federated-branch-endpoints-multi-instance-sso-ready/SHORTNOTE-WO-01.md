---
title: 'SHORTNOTE: Handoff WO-01 Endpoint Registry'
description: 'Catatan handoff ke Dev Agent untuk WO-01: kondisi kode pasca-destroy, prasyarat internal/migration, urutan eksekusi, jebakan, dan DoD. Termasuk fakta bahwa tabel migrations/migration_targets sudah ada di DB.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
---

# Shortnote Handoff — WO-01 Endpoint Registry

**Untuk:** Dev Agent
**Sumber:** `SPEC.md`, `WO-01-endpoint-registry.md`
**Mulai dari commit:** `1f5a9b2` (setelah destroy topic multi-DB)

---

## 0. Kondisi kode saat ini (sudah diverifikasi manager)

```
BUILD_OK
ok  internal/events, internal/pluginloader, internal/plugins, internal/secrets,
    plugins/auth/model, plugins/pages, plugins/plugin_registry
```

| Ada | Hilang |
|---|---|
| `plugins/auth` (+ JWT, middleware, migrations) | `internal/migration` + `console migrate` |
| `internal/secrets` (AES-GCM) | `internal/db/registry.go` |
| `internal/uuid` (UUIDv7) | `plugins/business*` |
| `internal/console` (CLI framework) | `cmd/business` |

**Baca dulu:** yang dibongkar adalah lapisan multi-DB, **bukan** fondasi aplikasi. Jangan panik melihat folder kosong; bangun di atas yang tersisa.

---

## 1. Jebakan #1 (paling penting) — tabel `migrations` SUDAH ADA di DB

Saat destroy, kita **tidak** drop database (keputusan user). Akibatnya:

```
bbh_data_board      → sisa tabel: admins, members, migrations, migration_targets
bbh_business        → sisa tabel: migrations, migration_targets
bbh_cbg_jktbaru     → sisa tabel: migrations, migration_targets
bbh_cbg_smg         → sisa tabel: migrations, migration_targets
```

Artinya saat kamu menulis ulang `internal/migration`:

- Migrator baru **harus kompatibel** dengan bentuk tabel yang sudah ada, **atau** memakai `CREATE TABLE IF NOT EXISTS` dengan kolom yang cocok.
- **Jangan** mengasumsikan tabel kosong lalu melempar `CREATE TABLE` tanpa `IF NOT EXISTS` → error `relation already exists` pada DB yang sudah dipakai.
- **Cek dulu bentuk kolomnya** sebelum menulis kode:
  ```bash
  docker compose exec -T postgres psql -U postgres -d bbh_data_board -c '\d migrations'
  docker compose exec -T postgres psql -U postgres -d bbh_data_board -c '\d migration_targets'
  ```
- Baris lama mungkin masih ada (target `business`, `business_generator` dengan `dirty=false`). Bersihkan bila mengganggu, tapi **jangan** `DROP DATABASE`.

---

## 2. Jebakan #2 — `bin/` dan build

`bin/console` **tidak** ikut ter-update oleh `go build ./...`. Sudah 2× menyebabkan kebingungan ("fitur tidak jalan" padahal kode benar). Selalu:

```bash
docker compose exec -T app sh -c 'cd /src && go build -o bin/console ./cmd/console'
```

`bin/business` sudah dihapus (tidak dipakai lagi di arsitektur federasi). `bin/server` lawas (18:35) — rebuild bila perlu uji server.

---

## 3. Jebakan #3 — rahasia tidak lewat argv

Jangan menambah flag `--token`. Token hanya via `--token-stdin` atau prompt no-echo. Ini kontrak yang sudah dibayar mahal di tema lama (WO-11 → WO-12: prompt tidak terlihat karena `/dev/tty` dibuka `O_RDONLY`).

**Kalau kamu membuat prompt interaktif**, buka `/dev/tty` dengan `os.OpenFile("/dev/tty", os.O_RDWR, 0)` — **bukan** `os.Open`. Dan **jangan telan error tulis**; kembalikan.

---

## 4. Jebakan #4 — partial update jangan reset status

Komunitas bug klasik: handler `PUT` menyalin struct penuh ke upsert, `status` yang tidak dikirim jadi kosong, lalu diisi default `active`. **Server yang sengaja disuspend hidup lagi diam-diam.**

Di WO ini: pakai pointer (`*string`, `*int`, `*bool`) dan `cmd.Flags().Changed(...)`; jangan `if field != ""`.

---

## 5. Urutan eksekusi yang disarankan

```
A. Cek bentuk tabel migrations/migration_targets (psql \d)
B. Tulis internal/migration minimal  → migrate up/down/list --plugin
C. Uji: migrate up --plugin auth di DB baru yang kosong; lalu down
D. Buat plugin `endpoint` + 1 migrasi (endpoints + endpoint_audit + seed local)
E. migrate up --plugin endpoint  → verifikasi tabel + seed
F. CLI: add → list → test → use → active → rm (guard)
G. Admin API (partial PUT, tanpa token di response)
H. Audit hook (request_id, tanpa body/token)
I. go build/vet/test + rebuild bin/console
```

**Jangan mengerjakan B..I sekaligus.** Selesaikan B+C, laporkan, baru lanjut. Bagian A salah bentuk = semua di atasnya salah.

---

## 6. Kontrak teknis ringkas

**Skema** — lihat `WO-01` Bagian B. Poin wajib:
- `key` UNIQUE, immutable
- `token_enc` = envelope AES-GCM (`internal/secrets.EncryptBlob`), **NULL** bila `auth_kind='none'`
- seed `local` (`is_local=TRUE`) selalu ada
- **nol `CREATE TABLE` di Go**

**Endpoint aktif:** rekomendasi kolom `is_active` di tabel (bukan file state) — hanya satu baris aktif, dikelola dalam transaksi. Alasan: state tidak hilang antar proses, dan `endpoint active` konsisten dari CLI maupun server.

**`test`:** panggil `<base_url>/healthz`, fallback `/`. Timeout dari kolom `timeout_ms`. Laporkan sebab terklasifikasi (tabel pemetaan ada di WO-01 Bagian C), bukan `err.Error()` mentah.

**`rm` guard:** tolak `is_local` **dan** endpoint aktif.

---

## 7. Definition of Done (buktikan, jangan klaim)

1. `migrate up --plugin endpoint` → tabel + seed `local` ada (lampirkan output `\d endpoints`).
2. `endpoint add` → lampirkan query yang menunjukkan `token_enc` **bukan** plaintext.
3. `endpoint test` dua arah: host hidup (durasi) **dan** host mati (sebab aksiabel + exit 1).
4. `endpoint rm --key local` → ditolak; `endpoint rm --key <aktif>` → ditolak.
5. `grep -rn 'CREATE TABLE' internal/ plugins/ --include=*.go` → **0 hits**.
6. `go build ./... && go vet ./... && go test ./...` → lulus (tempel output).
7. Response API & output CLI: **grep token → 0 hits**.
8. Rebuild `bin/console` dari source terbaru.

**Aturan kejujuran:** bila ada butir yang tidak bisa kamu uji (mis. belum ada instance cabang untuk `endpoint test`), **tulis "belum terbukti"** beserta alasannya. Jangan tandai selesai. (Pelajaran mahal dari tema sebelumnya: satu AC ditandai "sesuai algoritma" tanpa dijalankan, dan bug-nya baru ketahuan dari user.)

---

## 8. Yang belum dikerjakan (jangan sentuh di WO ini)

- Proxy/switch per request via header `X-Branch-Endpoint` → **WO-02**
- SSO lintas instance → **fase 2**
- `endpoint update` (CLI) → WO terpisah (tapi API `PUT` sudah partial di sini)
- Dashboard pemilihan cabang
- Sinkronisasi data antar cabang
