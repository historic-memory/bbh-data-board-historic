---
title: konteks
description: "Folder induk sejajar bbh-branch berisi instance per cabang (selain bali)"
status: complete
created: "2026-11-24"
updated: "2026-11-24"
---
# konteks

## Keputusan user (24 Nov 2026)
- Setiap cabang membutuhkan **instance/instalasi sendiri** (bukan 1 DB
  untuk semua cabang — tabel hsh/sec/wh tidak punya branch_key).
- Struktur deployment: folder induk sejajar lainnya:
  `~/workspaces/bbh-branch/` — **isinya folder per cabang, SELAIN bali**.
- Bali tetap memakai instance yang sudah berjalan (bbh-data-board, local /
  yang nanti dipasang nginx `bbh.rolldev.my.id`).

## Rencana struktur
```
~/workspaces/
├── bbh-data-board/          # instance BALI (pusat) — sudah ada
├── bbh-dummy-data/          # repo data dummy 5 cabang — sudah ada
└── bbh-branch/              # BARU — folder induk cabang non-bali
    ├── jogja/
    ├── semarang/
    ├── lombok/
    └── java-children/
```
Masing-masing folder cabang = project deployment siap compose:
- `.env` sendiri: `INSTANCE_KEY=<key>`, `INSTANCE_LABEL=<nama>`,
  `APP_PORT` berbeda, DB berbeda (`DB_NAME=bbh_<key>`),
  CORS domain cabang, `SSO_*` bila pakai portal.
- `docker-compose.yml` sendiri (copy dari bbh-data-board dengan nama
  project unik: `bbh-jogja`, dst — jangan tabrakan container/port/volume).
- DB postgres sendiri per cabang (kalau share satu postgres, beda DB
  nama; kalau per-cabang container postgres sendiri, pastikan host port
  beda).

## Fakta teknis dasar (dari repo bbh-data-board)
- Kolom `branch_key` hanya di: facilities, residents, visit_visits,
  ingest_sessions → hsh/sec/wh TIDAK bisa multi-cabang dalam 1 DB.
- `INSTANCE_KEY` dibaca plugin federation (default `local`).
- Key cabang: `bali`, `jogja`, `semarang`, `lombok`, `java-children`.
- Prasyarat tiap instance setelah deploy:
  1. `console maintenance clean-data --yes --reseed` (kategori hsh/sec).
  2. Daftar 3 gudang dummy cabang (fail-closed) — nama dari
     `branches/<key>.yaml` di repo bbh-dummy-data.
  3. Import folder dummy cabang itu Jan→Des.
  4. Akun admin dibuat (login pertama).

## Pertanyaan terbuka (SUDAH TERJAWAB 24 Nov 2026)
1. ~~Semua 4 cabang di host yang sama (1 laptop/VPS) atau tersebar?~~
   → **Semua di laptop ini** (dikerjakan di WO-01).
2. ~~Domain akses per cabang (mis. `jogja.rolldev.my.id`) — perlu atau
   cukup port?~~ → **Cukup port dulu** (8101–8104); domain menyusul.
3. ~~Port APP_PORT per cabang — urutkan?~~ → **8101 jogja, 8102 semarang,
   8103 lombok, 8104 java-children** (persis seperti usulan).

## Catatan
- Topic ini untuk PLAN + WO pembuatan scaffolding deployment; eksekusi
  = Dev Agent.
- Tidak menyentuh bbh-dummy-data (topic 1795513709400) dan instance
  bali yang berjalan.

## Status Eksekusi (24 Nov 2026) — WO-01 COMPLETE

Keempat cabang **hidup**: jogja 8101, semarang 8102, lombok 8103,
java-children 8104 (`/healthz` 200, login admin lokal 200). Endpoint
terdaftar di bali + probe Tailscale 200. 3 gudang dummy/cabang terdaftar.
Import dummy Jan→Des selesai (360 baris/segmen/cabang, 0 rejected).
SSO portal-token terverifikasi (pending 403 → approve → 200).

**Dua hal yang MENYIMPANG dari rencana awal (perlu diketahui):**
1. **Deployment butuh `docker compose` ≥ 2.24.** Override memakai tag
   `!reset`/`!override`; Compose v2.5.0 (bawaan sistem) tidak mendukung
   dan akan bentrok port. Dipasang **v2.32.4** di level user
   (`~/.docker/cli-plugins/docker-compose`, tanpa sudo, reversible).
   Ini prasyarat operasional semua `docker compose` milik user ini.
2. **Tiap cabang butuh baris `endpoints` miliknya** (`console endpoint add
   --key <key>`). Karena `branch_key` default = `INSTANCE_KEY` dan
   divalidasi ke `endpoints.key`, sedangkan migrasi hanya seed `local`.
   Tanpa baris ini semua tulis/baca resident & visit = 400
   `ErrUnknownBranch`. Ini data, bukan perubahan source code.

README operasional: `~/workspaces/bbh-branch/README.md`.

## Mekanisme login cabang via portal (fakta plugin portalsso)
- Login TETAP di bali (pusat): `POST /api/admin/portal-token`
  (permission `endpoint.write`) → token identity HS256
  (iss=bali, audience=endpoint cabang).
- Cabang memverifikasi token LOKAL via `SSO_SHARED_SECRET`
  (shared key, WAJIB sama di bali + semua cabang; TIDAK boleh sama
  dengan APP_KEY — fail-closed `ErrSharedSecretReused`).
- Identity (iss+sub) dicocokkan ke tabel `members` cabang
  (`MembershipResolver.Resolve`) — tanpa callback ke pusat.
- Pertama kali → auto-provision `status=pending`, `origin=sso`
  (D-18 option B) dan MASIH ditolak (403) sampai admin cabang
  approve: `GET /api/admin/sso-enrollments` → `POST /:id/approve`
  (butuh `group.write`).
- Password login lokal di cabang tetap ada sebagai fallback admin.

## Prasyarat SSO per deployment (dicatat agar WO lengkap)
1. `SSO_SHARED_SECRET` sama di semua instance (bali + 4 cabang),
   beda dari APP_KEY masing-masing.
2. Endpoint cabang didaftarkan di bali (plugin `endpoint`, Key =
   jogja/semarang/lombok/java-children; BaseURL sesuai host/domain).
3. Admin cabang (local) cukup 1 untuk approve enrollment pertama.
4. Urutan pemasangan cabang:
   scaffold compose → seed (clean-data --yes --reseed) → buat admin
   → daftar gudang dummy cabang → approve enrollment user portal →
   import dummy Jan→Des.
