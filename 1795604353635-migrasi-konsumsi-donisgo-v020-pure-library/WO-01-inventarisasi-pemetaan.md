---
title: WO-01 — Inventarisasi & pemetaan (read-only)
description: "Inventaris lengkap internal/* + pemetaan ke library donisgo v0.2.0, diff plugin_registry, cek env port, baseline swagger/binari. TANPA perubahan kode."
status: pending
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, wo-01, read-only]
---

# WO-01 — Inventarisasi & pemetaan (read-only)

Ref: `spec.md` §Pemetaan internal/* → target, §Perubahan perilaku. Prasyarat: —.

## Lingkup

WO ini **TIDAK mengubah kode sumber apa pun**. Outputnya adalah data untuk WO-02 (dan keputusan owner bila ada temuan divergensi).

## AC

- **AC-1 — Inventaris paket `internal/`**: daftar lengkap paket di `internal/` (nama, jumlah file, satu kalimat fungsi). Tabel pemetaan di SPEC diperiksa satu per satu; setiap baris mendapat status ✅ cocok-library / ⚠️ ada-divergensi / ❌ milik-app. Bukti: tabel di section `## Hasil Inventaris (Dev)` dokumen ini.
- **AC-2 — Diff `plugins/plugin_registry`**: bandingkan `plugins/plugin_registry/{errors,registry}.go` + test dengan `github.com/rolldone/donisgo/plugins/plugin_registry@v0.2.0` (fetch via `go mod download` ke module cache atau raw.githubusercontent). Bila identik → catat "swap aman". Bila ada perbedaan (termasuk komentar/konstanta ekstra) → **BERHENTI, lapor owner** (D-3 fail-loud); jangan tukar diam-diam. Bukti: output diff (atau kalimat "tidak ada perbedaan") + hash kedua file.
- **AC-3 — Nasib `internal/admin`**: telusuri isi `internal/admin/services` (direferensikan DONISGO.md sebagai "business services"). Bila kosong/stub → catat untuk dihapus di WO-02. Bila ada kode bisnis → daftar dependensinya (plugin mana yang memakai) + usulan penempatan, **tunggu keputusan owner sebelum lanjut** (O-A di SPEC).
- **AC-4 — Env port**: dari sumber `github.com/rolldone/donisgo/app@v0.2.0`, temukan env yang dibaca untuk host/port (`APP_HOST`/`APP_PORT` vs `PORT`). Jawab tegas: env apa yang dipakai v0.2.0, apakah nama lama masih didukung, dan apa yang harus diubah di `.env` bbh / `docker-compose.yml` / webapp `VITE_API_TARGET`. Bukti: kutipan kode sumber (file:baris) di dok WO.
- **AC-5 — Pemakaian API `auth` package-level**: grep semua pemanggil `auth.Sign*`/`auth.Parse*`/`auth.GenerateAccessTokenWithLevel`/`auth.ParseAccessTokenClaims`/`auth.AccessExpirySeconds`/`auth.RefreshExpirySeconds` di kode bbh. Output: daftar call-site per plugin (untuk dijadikan bahan WO-03). Bukti: tabel call-site.
- **AC-6 — Pemakaian API lain yang berubah**: cek call-site `events.*` (package-level vs `NewBus`), `keydb.*`, `db.GetGormDB()`, `storage.*` package-level, `console.*` (root/migrate). Klasifikasi: tetap pakai library package-level (deprecated tapi aman) vs harus di-inject di WO-02/03. Bukti: tabel.
- **AC-7 — Baseline pra-migrasi**: simpan di dok WO — (a) `go version -m bin/server bin/console` (stamp VCS saat ini), (b) jumlah path/ops swagger saat ini dari `docs/swagger.json`, (c) hasil `go build ./... && go vet ./... && go test ./...` sebagai kondisi awal (catat test yang memang merah pre-existing, mis. test resident vs data dev — topic `1793687545424` T-1), (d) `docker compose version` di host.
- **AC-8 — Laporan keputusan**: rangkum semua temuan ⚠️/❌ + O-A/O-B ke chat; bila tidak ada yang perlu keputusan, tulis "tidak ada blocker" eksplisit.

## Non-tujuan
Edit file Go apa pun, ubah go.mod, jalankan migrasi DB, commit kode.

## Catatan Implementasi (Dev)

(dikosongkan — diisi dev bertahap sesuai aturan log implementasi)

## Review Manager

(dikosongkan)
