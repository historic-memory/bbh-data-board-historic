---
title: "WO-03: Verifikasi & dokumentasi deploy"
description: "Verifikasi menyeluruh AC-1..AC-7 pada image hasil WO-01, plus dokumentasi deploy: env wajib, urutan migrasi, rollback, catatan CWD."
status: pending
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - deploy
  - verification
  - docs
---

# WO-03: Verifikasi & dokumentasi deploy

## Referensi
- SPEC: AC-1..AC-12, GAP-4, GAP-5.
- Hasil **WO-01** (image) & **WO-02** (audit).

## Tugas
1. **Verifikasi end-to-end** pada container hasil `Dockerfile.prod`:
   | Cek | Target |
   |---|---|
   | `GET /app/` | 200 |
   | `GET /app/residents` (deep link) | 200 `index.html` |
   | `GET /app/assets/<file>` | 200, `Content-Type` benar |
   | `GET /api/admin/residents` tanpa token | 401 JSON |
   | `GET /api/tidak-ada` | 404 JSON |
   | `GET /healthz` | 200 |
   | User container | non-root |
   | `command -v node` | tidak ada |
   | Ukuran image | tercatat |
   | Entrypoint tanpa `dist/index.html` | exit ≠ 0 |
2. **Dokumentasi** (di `README.md` root atau `docs/DEPLOY.md` baru):
   - Perintah build & run image.
   - Env wajib: `APP_KEY`, `DB_*`, `KEYDB_*`, `PORT`, `INSTANCE_KEY`, `CORS_ALLOWED_ORIGINS`,
     `HOMEPAGE_URL` (opsional).
   - **Urutan deploy**: start → `console migrate` (eksplisit, bukan otomatis) → healthcheck.
   - **Catatan CWD**: `spaDir` relatif → server harus dijalankan dengan `WORKDIR /app`
     (sudah di-set di image).
   - Rollback: tag image sebelumnya + catatan migrasi tidak otomatis di-rollback.
3. Update SPEC/`work-order.md`: isi status + angka nyata (ukuran image, hasil curl).

## Kriteria Selesai
- AC-1..AC-12 seluruhnya terbukti dengan output nyata di laporan.
- Dokumentasi deploy tersedia & akurat (tidak ada klaim tanpa dicek).
- Dev workflow terbukti tidak berubah (`docker compose up -d` + `npm run dev` tetap jalan).
