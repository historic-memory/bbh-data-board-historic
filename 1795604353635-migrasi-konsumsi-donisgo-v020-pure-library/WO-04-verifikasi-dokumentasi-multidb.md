---
title: WO-04 — Verifikasi & dokumentasi + re-review WO multi-DB (D-5)
description: "Swagger re-baseline pasca module rename, Docker/compose, .env.example, README repoint + DONISGO.md di-usangkan, Bruno regresi RBAC, re-review strategi WO multi-DB (1792765876397), housekeeping."
status: pending
created: "2026-11-25"
updated: "2026-11-25"
tags: [donisgo, migrasi, wo-04, verifikasi]
---

# WO-04 — Verifikasi & dokumentasi + re-review WO multi-DB (D-5)

Ref: `spec.md` §Perubahan perilaku 4–6, D-5, D-6. Prasyarat: WO-03 complete.

## Lingkup

**A — Swagger re-baseline**
- Regenerasi `docs/swagger` via `build.sh` di container `app` (lingkungan kanonik; jangan host) — anotasi mengutip `model.*` jadi `github_com_akasatechdev_bbh_data_board_plugins_...`.
- Angka paritas WAJIB diukur ulang (gotcha terdokumentasi: angka handoff sering usang; filter substring ikut menangkap path lain).
- `build.sh --verify-docs` exit 0, byte-identik setelah re-baseline; commit hash manifest baru tercatat.
- JEBAKAN swag berulang: file yang mengutip `model.X` di godoc wajib import paket model + blank-var; perbaiki semua yang kena (pola resident/warehouse/ingest).

**B — Docker & compose**
- Dockerfile/Dockerfile.runtime/compose ditinjau: bila ada path `internal/` yang di-COPY/referenced → disesuaikan; env port sesuai temuan WO-01; `TRUSTED_PROXIES` (default trust-none) masuk `.env.example` + dokumentasi (bbh di-compose TANPA reverse proxy → default aman, catat implikasi bila nanti di-deploy di belakang proxy).

**C — Dokumentasi**
- README root: repoint dokumentasi aplikasi ala template (quickstart, env, migrate, plugin dev) + catatan ekosistem (donisgo library di-pin; template = referensi struktur).
- `DONISGO.md`: tambahkan banner USANG di atas (konten monorepo tidak lagi valid; arahkan ke pkg.go.dev + README donisgo-template). Jangan dihapus total (masih ada jejak sejarah) — cukup header.
- `.env.example`: sinkron dengan env library v0.2.0 (port/TRUSTED_PROXIES/JWT legacy note).

**D — Bruno regresi**
- `bru run --env Local --tags=smoke` PASS; sampling folder domain (dashboard/secondary/warehouse visit) sesuai konvensi read-only; RBAC viewer/limited sampling (200/403); token di-refresh dulu (gotcha kedaluwarsa).

**E — Re-review WO multi-DB (D-5)**
- Baca ulang topic `1792765876397` (SPEC complete, WO draft) — target lama `internal/console/migrate.go` & `internal/db/gorm.go` kini milik library donisgo.
- Tulis **rekomendasi tertulis** di dok ini: opsi (1) up-stream request/kontribusi ke donisgo; (2) fork kecil di sisi app (mis. engine migrate bbh sendiri di `cmd/console`); (3) tunda hingga library menyediakan multi-conn. Plus daftar gap spesifik bbh (per-branch DB `bbh_<key>`).
- **KEPUTUSAN FINAL diambil owner** — Manager hanya menyajikan opsi + usulan.

**F — Housekeeping**
- `.env.example` JWT 900→7200 (sisa WO-04 topic `1793687545424`) — ikut commit di sini bila masih uncommitted.
- Cek binari `bin/` (gitignored) tidak ikut commit; `git add` path spesifik saja (gotcha insiden 80MB).

## AC

- **AC-1**: `build.sh --verify-docs` exit 0; manifest baru; angka path/ops terukur & tercatat.
- **AC-2**: build image `app` sukses; server di container boot; `/healthz` 200 dari host.
- **AC-3**: `.env.example` berisi TRUSTED_PROXIES + port yang benar; README & DONISGO.md sesuai C.
- **AC-4**: Bruno smoke PASS (angka req/test tercatat); RBAC sampling lolos.
- **AC-5**: re-review multi-DB tertulis lengkap (3 opsi + gap + usulan); status di dok ini "menunggu keputusan owner".
- **AC-6**: housekeeping F selesai (JWT env & hygiene git).
- **AC-7**: data uji & scaffolding bersih; tidak ada token/PII di-commit.

## Catatan Implementasi (Dev)

(dikosongkan)

## Review Manager

(dikosongkan)
