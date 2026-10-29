---
title: notes
description: "Catatan awal (draft) modul baru: list kebutuhan rumah tangga per cabang BBH."
status: create
created: "2026-10-29"
---
# Notes — Kebutuhan Rumah Tangga

## Konteks
- Modul **baru**: pencatatan **list kebutuhan rumah tangga** (mis. perlengkapan dapur, alat kebersihan, perabot, dsb.) untuk cabang-cabang BBH.
- Status: **baru dibuka** (2026-10-29). Data mentah & SPEC **menyusul** dari owner.

## Yang sudah diputuskan
- Berdiri sebagai **topic terpisah** dari "Kebutuhan Elektronik" (keputusan owner).
- Akan dibangun sebagai **plugin baru** di `plugins/<id>/` (aturan user: fitur baru = plugin, tidak menaruh kode fitur di `internal/`).

## Usulan (belum dikunci)
- **Usulan plugin id:** `plugins/needs` (satu plugin untuk kedua list, dibedakan lewat `category: electronics | household`).
  Alternatif: plugin terpisah bila struktur kedua list ternyata berbeda jauh.
- Gap yang perlu dijawab saat SPEC: sumber data (CSV/Notion?), apakah per-cabang (pakai `endpoints.key`/`INSTANCE_KEY`, bukan tabel `branches`), perlu quantity/harga/status pemenuhan, RBAC permission key, dukungan impor/template.

## Pertanyaan terbuka
- [ ] Bentuk data mentah (kolom apa saja) — menunggu owner.
- [ ] Apakah list ini sekadar catatan kebutuhan (wishlist) atau punya alur status (diajukan → disetujui → dibeli → diterima)?
- [ ] Satu plugin `plugins/needs` atau dua plugin terpisah?
- [ ] Kaitan dengan modul inventory/POS cabang (topic federated endpoints) — ada atau tidak.

## Langkah berikutnya
1. Owner menyiapkan data mentah.
2. Manager susun `spec.md` (keputusan desain + gap) lalu `work-order.md`.

