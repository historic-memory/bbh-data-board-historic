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
| WO-03 | Migrasi auth ke API instans (D-4) | complete (review Manager LULUS) | — | `c495e2c` | `e42314d` |
| WO-04 | Verifikasi & dokumentasi + re-review WO multi-DB (D-5) | complete (review Manager LULUS — topik TUNTAS) | — | `fafafd8` + `af73b4f` | `3490295` |

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

## Hasil WO-03 (2026-11-25, review Manager LULUS — kode `c495e2c`, dok `e42314d`)

- **Migrasi auth ke API instans tuntas**: 1 titik konstruksi `auth.New(auth.ConfigFromEnv())` di `plugins/auth/plugin.go:65` (dari `buildTokenAuth`); 5 call-site wrapper deprecated diganti method instance; grep wrapper di source = 0; signature publik tidak berubah → 9 plugin lain tidak tersentuh.
- **Deviasi sadar SAH (adili Manager)**: `ConfigFromEnv()` library punya fallback insecure (`change-me-*`) → tanpa guard, AC-3 pasti gagal. Guard eksplisit (`AUTH_JWT_SECRET` & `JWT_ACCESS_SECRET` kosong → error) = perbaikan keamanan riil; env legacy tetap didukung (`TestBuildTokenAuth_LegacyAccessSecret`).
- **Desain penyimpanan instance SETUJU (adili Manager)**: `SetTokenAuth`/`TokenAuth` di paket `services` alih-alih parameter — menghindari puluhan call-site di luar lingkup.
- Test 4/4 PASS (fail-closed ×2 + isolasi instance + legacy); runtime login root 200 via instance baru; token lama pre-change tetap valid; RBAC regression (viewer 200/403) lulus; CLI auth jalan.
- **Temuan baru → WO-04**: gofmt debt 27 file pra-ada (sisa rewrite WO-02) → commit `style: gofmt` terpisah SEBELUM regen swagger.

## Hasil WO-04 (2026-11-25, review Manager LULUS — kode `fafafd8`+`af73b4f`, dok `3490295`)

- **Swagger re-baseline**: 141 paths/209 ops/248 defs (identik baseline, nol regresi), 0 sisa `rolldone` di definitions; regen di container; idempotent byte-identik.
- **Housekeeping**: gofmt 27 file → `gofmt -l` = 0 (commit terpisah `fafafd8`); JWT env sudah 7200.
- **Docs**: `.env.example` TRUSTED_PROXIES + APP_PORT diperjelas; README ekosistem/struktur tanpa internal; `DONISGO.md` banner USANG + koreksi internal/admin.
- **Bruno**: smoke 13 req/36 test PASS; RBAC viewer 200/403; limited group dashboard 12/12; data uji dibersihkan.
- **D-5 DITUTUP BEDA DARI RENCANA (temuan besar dev, verifikasi Manager)**: topic `1792765876397` **sudah di-destroy owner** (bukti SPEC federasi `1793013271539:14` — arah multi-DB terpusat dibatalkan, diganti federasi instance). Keputusan §E = **Opsi 3 (tunda) diterima**; gap A–E terdokumentasi di dok WO; draft issue upstream H-3 siap kirim.
- Jebakan swag baru terdokumentasi: `analysis_apply.go` (blank-var + nama tipe baru), `record_write.go:181` (path lama). Gotcha Bruno: `{{process.env.BBH_TOKEN}}` tak terbaca bru 4.2.1 → wajib `--env-var token=...`.
- **MIGRASI KE DONISGO v0.2.0 PURE LIBRARY SELESAI** — rekap kode: `e99477d` (inti 244 file), `c495e2c` (auth instans), `fafafd8` (gofmt), `af73b4f` (verifikasi/docs). Semua tanpa push.

## Penutup Topik (Manager, 2026-11-25)

MIGRASI TUNTAS — semua 4 WO complete & lulus review. Bukti akhir: `go.mod` = `github.com/AkasaTechDev/bbh-data-board` + `require donisgo v0.2.0`; nol import `donisgo/internal` di source; `internal/` + `plugins/plugin_registry/` terhapus; serving SPA via pages (varian A); `/healthz` plugin; migrate `--plugin core` aman (dir core kosong); auth instans fail-closed; swagger re-baseline; Bruno/RBAC hijau. **Keputusan owner pasca-topik (menunggu)**: (a) push 4 commit ke `origin/master`; (b) kirim draft issue H-3 ke `donisgo-template`; (c) konfirmasi close topic; (d) patch kecil Bruno env (opsional, tanpa WO).

- **WO-01** — Tanpa perubahan kode. Output = tabel pemetaan terverifikasi + laporan divergensi (`plugin_registry`, `internal/admin`) + jawaban env port (`APP_PORT` vs `PORT` di sumber `app` v0.2.0) + baseline swagger/binari. Bila D-3 fail-loud terpicu → BERHENTI, lapor owner.
- **WO-02** — Satu commit atomik: `go.mod` rename + `require donisgo v0.2.0`, rewrite ~104 import, hapus `internal/*` yang terganti, bootstrap `cmd/server` (registrasi 13 plugin eksplisit + `EnsurePluginLast(pages)`), `cmd/console` (`RegisterAdditionalPlugins` + generator `plugin new` dari template), hapus `internal/pluginloader`. Gates: build/vet/test, boot server, migrate idempotent, CLI plugin jalan, Bruno smoke.
- **WO-03** — `plugins/auth` pakai `auth.New(auth.ConfigFromEnv())` satu titik; hapus wrapper deprecated dari kode bbh; test isolasi + fail-closed.
- **WO-04** — Swagger re-baseline (module path baru) + `build.sh --verify-docs`; Docker/compose env sesuai temuan WO-01; `.env.example` (TRUSTED_PROXIES, port); README repoint + `DONISGO.md` di-usangkan; re-review multi-DB (rekomendasi tertulis di dok WO, keputusan owner).

## Penutup topik

Semua WO complete → Manager cek `historic show 1795604353635 --json` status SEMUA file → tulis section "Penutup Topik" di dok ini → `historic save`. Dev diingatkan pola berulang: sering lupa menutup master index.
