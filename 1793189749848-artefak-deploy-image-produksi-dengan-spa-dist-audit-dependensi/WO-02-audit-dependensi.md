---
title: "WO-02: Klasifikasi & perbaikan aman audit dependensi"
description: "Evaluasi 10 advisory npm pada sub_app/webapp, terapkan perbaikan yang tidak breaking, dan dokumentasikan advisory yang butuh major bump. Menutup H-3."
status: pending
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - security
  - frontend
---

# WO-02: Klasifikasi & perbaikan aman audit dependensi (H-3)

## Referensi
- SPEC: A-12, AC-8, GAP-3.
- `sub_app/webapp/package.json` (`axios ^1.7`, `react ^19`, `react-router-dom ^7`,
  `vite ^6`, `tailwindcss ^4.3.3`, `@vitejs/plugin-react ^4.4.0`).

## Tugas
1. Jalankan `npm audit --json` di `sub_app/webapp`; simpan ringkasan (advisory count per
   paket + severity + `range` + `fixAvailable`).
2. **Klasifikasi tiap advisory** ke salah satu:
   - **A — aman diperbaiki sekarang** (`npm audit fix` tanpa `--force`, atau bump patch/minor).
   - **B — butuh major bump** (vite 7 / router 8 / react major) → **keputusan terpisah**, catat
     risiko & rekomendasi.
   - **C — tidak relevan runtime** (hanya dev-time/build) → catat alasannya.
3. Terapkan hanya kategori **A**; jalankan `npm run build` + `tsc -b` untuk memastikan tidak
   merusak build, lalu `bru run --env Local --tags=smoke` bila server hidup (opsional).
4. Tulis laporan di WO ini: tabel `paket | severity | fixAvailable | kategori | tindakan`.
5. Update `package-lock.json` bila ada perubahan.

## Larangan
- **JANGAN** `npm audit fix --force` (menurunkan/mengganti versi paket inti → risiko breaking).
- Jangan menaikkan major framework tanpa keputusan owner.

## Kriteria Selesai
- AC-8: setiap advisory punya kategori & alasan; perbaikan kategori A diterapkan.
- `npm run build` (HOST) lulus; `npm audit` dijalankan ulang & angkanya dicatat sebelum/sesudah.
- Tabel hasil di WO ini + rekomendasi lanjutan untuk kategori B.
