---
title: wo 01 branch scaffold
description: 'Scaffold 4 instance cabang non-bali di ~/workspaces/bbh-branch (host: laptop ini)'
status: planned
created: "2026-11-24"
updated: "2026-11-24"
---
# wo 01 branch scaffold

## Ringkasan
Scaffold **4 instance** cabang (jogja, semarang, lombok, java-children) di
`~/workspaces/bbh-branch/`, semua di **host laptop ini** (keputusan user),
akses via **port** (tanpa domain dulu), container/compose project unik.

## Keputusan user (TERKUNCI 24 Nov 2026)
1. Host: laptop ini (semua cabang).
2. Akses: port saja dulu (domain menyusul bila perlu).
3. Port: jogja **8101**, semarang **8102**, lombok **8103**,
   java-children **8104** (host APP_PORT → container 8080).

## Tugas
1. [ ] Buat `~/workspaces/bbh-branch/` + 4 folder cabang
       (`jogja`, `semarang`, `lombok`, `java-children`).
2. [ ] Tiap folder cabang = **CLONE repo aplikasi**
       `git clone https://github.com/AkasaTechDev/bbh-data-board.git <key>`
       (keputusan user 24 Nov 2026: "pull repo tiap tiap branch, gampang
       update") — di dalam clone buat file LOKAL gitignored:
       - `.env` per cabang (lihat tugas 3)
       - `docker-compose.override.yml` (container_name prefix `bbh-<key>-`,
         APP_PORT host 8101-8104, port postgres/keydb unik, volume unik)
       Pastikan:
       - `container_name` unik (prefix `bbh-<key>-`)
       - `name:` compose project per cabang
       - host port APP_PORT: 8101/8102/8103/8104
       - postgres host port TIDAK clash (mis. 15441/15442/15443/15444
         atau pakai `expose` tanpa host port bila tak perlu akses luar)
       - keydb host port TIDAK clash (mis. 16381/16382/16383/16384)
       - `DB_NAME=bbh_<key>` (kalau memakai 1 container postgres share,
         pastikan database berbeda; pilih 1 postgres per cabang agar
         aman — jangan share)
3. [ ] Tiap `.env` cabang:
       - `INSTANCE_KEY=<key>` (jogja/semarang/lombok/java-children)
       - `INSTANCE_LABEL=<Nama Baby Home>`
       - `APP_PORT=<port>` (di atas)
       - `DB_*` db berbeda, `KEYDB_*` berbeda
       - `APP_URL=http://localhost:<port>` (akses via port dulu)
       - `CORS_ALLOWED_ORIGINS` sesuai port cabang (+ Tailscale bila
         ingin akses dari device lain)
       - `AUTH_JWT_SECRET`, `APP_KEY`, `INGEST_*` nilai BARU per cabang
         (JANGAN copy kredensial bali)
       - `SSO_SHARED_SECRET` = nilai yang SAMA dengan bali (jangan sama
         dengan APP_KEY cabang; fail-closed)
4. [ ] Bootstrap tiap instance:
       - `docker compose up -d` → server jalan
       - `docker compose exec app go run ./cmd/console` bila perlu
       - `console maintenance clean-data --yes --reseed`
       - buat admin local pertama (cara sesuai CLI/console yang ada —
         verifikasi dari kode; jangan asal)
       - cek `/healthz` 200 & login admin 200
5. [ ] Registrasi endpoint cabang di BALI (plugin endpoint):
       - Key = jogja/semarang/lombok/java-children
       - BaseURL = `http://100.125.175.34:<port>` (Tailscale, bukan
         localhost — federation proxy jalan dari proses bali)
       - Verifikasi federation bisa meneruskan (health per endpoint).
6. [ ] Daftar 3 gudang dummy per cabang (sesuai
       `~/workspaces/bbh-dummy-data/branches/<key>.yaml`):
       via UI admin atau `POST /api/admin/wh/warehouses` (akun local).
7. [ ] (Opsional/dev-only) Import dummy cabang: folder
       `~/workspaces/bbh-dummy-data/data/<key>/2026-01-januari …`
       via `/import` masing-masing segmen, urut Jan→Des. Catat hasil
       counts per tabel.
8. [ ] Verifikasi SSO (bila diinginkan user): issue portal-token di
       bali untuk endpoint cabang → coba akses cabang → enrollment
       pending → approve oleh admin cabang → akses 200.

## Acceptance
- 4 cabang hidup di laptop ini, masing-masing:
  - `/healthz` 200 di port masing-masing (8101–8104)
  - `INSTANCE_KEY` benar (cek `/api` atau log)
  - DB terpisah per cabang (counts isolat)
  - login admin local 200
- Endpoint terdaftar di bali (4 endpoint) + status aktif.
- Gudang dummy cabang terdaftar.
- TIDAK ada perubahan source code aplikasi `bbh-data-board`
  (hanya config/deployment files); commit historik dok update.

## Catatan
- SSO mekanisme + prasyarat: lihat konteks.md bagian
  "Mekanisme login cabang via portal".
- Dummy data per cabang: topic 1795513709400 (repo bbh-dummy-data).
- Kalau RAM laptop terbatas, jalankan cabang satu per satu
  (`docker compose down` saat tidak dipakai) — catat di README folder.
