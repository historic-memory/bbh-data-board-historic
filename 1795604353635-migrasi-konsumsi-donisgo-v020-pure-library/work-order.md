---
title: work-order
description: "Master index Work Order migrasi bbh-data-board ke donisgo v0.2.0 pure library (bentuk donisgo-template, tanpa internal/)."
status: progress
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, work-order, master-index]
---

# Work Order — Migrasi bbh-data-board ke donisgo v0.2.0

Topic: `1795604353635` · SPEC: `spec.md` (draft → disetujui owner 2026-11-25) · Dibuat: 2026-11-25

## Prinsip

1. **Partial principle**: WO dibuat penuh hanya untuk pekerjaan yang sudah jelas; WO lanjutan dibuat setelah WO sebelumnya lulus. Di topic ini SEMUA 4 WO sudah terdefinisi di SPEC §Rencana Work Order, jadi dibuat penuh.
2. **Migrasi inti ATOMIK**: rename module + rewrite import + hapus `internal/` + cmd baru = satu commit — tidak boleh ada keadaan "setengah rename" yang build merah.
3. **Bukti wajib per WO**: setiap AC diberi bukti konkret (output command / screenshot / curl / header), bukan naratif.
4. **Aturan log implementasi (owner 2026-11-06, topic `1793711295286`)**: dev menulis `## Catatan Implementasi (Dev)` di dok WO terkait — entri per baris kunci `berkas:baris` + apa + kenapa (ref D-x/O-x/gotcha), ±10–20 entri, diisi bertahap per bagian, ikut laporan dev dan checklist review Manager.
5. Data uji & akun uji dibersihkan di akhir tiap WO; token Bruno di-refresh tiap restart server (`api-tests/.env`, gitignored).

## Tabel WO

| WO | Judul | Status | Prasyarat | Commit kode | Dok historic |
|---|---|---|---|---|---|
| WO-01 | Inventarisasi & pemetaan (read-only) | complete (lulus review; O-B disetujui) | — | (tanpa kode) | `38b9ee3` + review `cabe109` |
| WO-02 | Migrasi inti: module rename + hapus internal/ + cmd ala template | complete (review Manager LULUS) | — | `e99477d` | `500a0eb` + review `cabe109`→terbaru |
| WO-03 | Migrasi auth ke API instans (D-4) | pending (SIAP) | WO-02 | — | — |
| WO-04 | Verifikasi & dokumentasi + re-review WO multi-DB (D-5) | pending | WO-03 | — | — |

Urutan eksekusi: WO-01 → WO-02 → WO-03 → WO-04 (linear; tidak ada yang bisa paralel karena semuanya menyentuh tree yang sama).

## Hasil WO-01 (2026-11-25, review Manager LULUS — dok historic `38b9ee3`)

- 12 paket `internal/` / 31 file; TIDAK ada kode bisnis; `mail` 0 importir; `pluginloader` hapus; `console/plugin.go` pindah ke app.
- **Koreksi SPEC (H-7)**: importir `internal/` = **55 file** (bukan 104 — terinflasi worktree `.kilo`).
- O-A selesai: `internal/admin` fiktif (hanya koreksi 2 baris `DONISGO.md` di WO-04).
- AC-4: library hanya baca env **`PORT`** (via gin); compose sudah benar; tidak ada perubahan env.
- WO-03 lingkup kecil: 5 call-site `auth` / 2 file, semua di plugin `auth`.
- Baseline: swagger 141/209/248; build/vet OK; 1 test merah pre-existing (`TestProxyPath`, bukan regresi migrasi).
- **BLOCKER O-B (DISELESAIKAN, owner 2026-11-25: SETUJU)** — swap `plugins/plugin_registry` → library dieksekusi di WO-02.
- **7 hazard utk WO-02..04**: H-1 default `migrate --plugin` `all`→`core` + tanpa dir `migrations/` → **solusi WO-02: buat `migrations/postgres/` kosong (.gitkeep) + koreksi docs**; **H-2 `/healthz` kehilangan rumah** → WO-02 pindah ke plugin kecil/pages (konsumen endpoint/federation/vite/client.ts tak boleh putus); **H-3 🔴 `EnsurePluginLast` setelah `app.New` TIDAK me-register route (pola README template bugged) → WO-02 WAJIB varian A (pages terakhir di `Config.Plugins`) atau B (pre-register sebelum New)** + usulan issue upstream ke `donisgo-template` (WO-04); H-4 generator `plugin new` jadi `console/plugin.go` app; H-5 `TRUSTED_PROXIES` default trust-none (dampak nol, dokumentasi WO-04); H-6 swagger re-baseline di container (WO-04); H-7 koreksi 104→55 di SPEC (WO-04). **WO-02 SIAP EKSEKUSI.**

## Hasil WO-02 (2026-11-25, review Manager LULUS — kode `e99477d`, dok `500a0eb`)

- **Migrasi inti tuntas ATOMIK**: 244 file +955/−4218; `go.mod` = `github.com/AkasaTechDev/bbh-data-board` + `require donisgo v0.2.0`; grep `donisgo/internal` source = 0; `internal/` + `plugins/plugin_registry/` terhapus (O-B dieksekusi).
- **H-3 varian A**: `pages` elemen TERAKHIR `app.Config.Plugins` (`cmd/server/main.go:57-73`); larangan pola template terdokumentasi di komentar main.go. **H-2**: `/healthz` rumah baru `plugins/healthz` (200). **H-1**: `migrations/postgres/.gitkeep` + `migrate up` POLOS exit 0; docs dikoreksi (README/AGENT/DONISGO).
- **H-4**: generator `plugin new` → `console/` milik app (plugin.go + plugin_test.go).
- Runtime: `/healthz` 200, `/` html, `/app/`+aset 200, `/spa` 301, `/api/*` 404-JSON, API 401; login root 200; RBAC utuh; CLI sampling (auth/endpoint/hsh/sso/maintenance) jalan; idempoten `down/up/up` (0 applied ke-2); bookkeeping `migration_targets(core)=1`, `migrations=33` tanpa eksekusi ulang.
- Test suite penuh `go test -count=1`: hanya `TestProxyPath` FAIL = pre-existing baseline (bukan regresi).
- **AC-4 anomaly accepted**: pengosongan `residents`/`wh_*` terjadi SEBELUM WO-02 (bukti sentinel-resistance + churn + mothers masih 10); state data dev `residents=0` kini baseline resmi.
- Sisa untuk WO-04: swagger re-baseline (module path baru), `DONISGO.md` banner usang + koreksi `internal/admin` referensi, TRUSTED_PROXIES dokumentasi, issue upstream H-3 ke donisgo-template, re-review multi-DB.

## Ringkasan per WO (rincian AC di file masing-masing)

- **WO-01** — Tanpa perubahan kode. Output = tabel pemetaan terverifikasi + laporan divergensi (`plugin_registry`, `internal/admin`) + jawaban env port (`APP_PORT` vs `PORT` di sumber `app` v0.2.0) + baseline swagger/binari. Bila D-3 fail-loud terpicu → BERHENTI, lapor owner.
- **WO-02** — Satu commit atomik: `go.mod` rename + `require donisgo v0.2.0`, rewrite ~104 import, hapus `internal/*` yang terganti, bootstrap `cmd/server` (registrasi 13 plugin eksplisit + `EnsurePluginLast(pages)`), `cmd/console` (`RegisterAdditionalPlugins` + generator `plugin new` dari template), hapus `internal/pluginloader`. Gates: build/vet/test, boot server, migrate idempotent, CLI plugin jalan, Bruno smoke.
- **WO-03** — `plugins/auth` pakai `auth.New(auth.ConfigFromEnv())` satu titik; hapus wrapper deprecated dari kode bbh; test isolasi + fail-closed.
- **WO-04** — Swagger re-baseline (module path baru) + `build.sh --verify-docs`; Docker/compose env sesuai temuan WO-01; `.env.example` (TRUSTED_PROXIES, port); README repoint + `DONISGO.md` di-usangkan; re-review multi-DB (rekomendasi tertulis di dok WO, keputusan owner).

## Penutup topik

Semua WO complete → Manager cek `historic show 1795604353635 --json` status SEMUA file → tulis section "Penutup Topik" di dok ini → `historic save`. Dev diingatkan pola berulang: sering lupa menutup master index.
