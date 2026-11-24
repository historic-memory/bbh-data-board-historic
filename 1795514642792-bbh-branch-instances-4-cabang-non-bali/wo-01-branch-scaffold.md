---
title: wo 01 branch scaffold
description: 'Scaffold 4 instance cabang non-bali di ~/workspaces/bbh-branch (host: laptop ini)'
status: complete
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
1. [x] Buat `~/workspaces/bbh-branch/` + 4 folder cabang
       (`jogja`, `semarang`, `lombok`, `java-children`).
2. [x] Tiap folder cabang = **CLONE repo aplikasi**
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
3. [x] Tiap `.env` cabang:
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
         dengan `APP_KEY` cabang; fail-closed)
4. [x] Bootstrap tiap instance:
       - `docker compose up -d` → server jalan
       - `docker compose exec app go run ./cmd/console` bila perlu
       - `console maintenance clean-data --yes --reseed`
       - buat admin local pertama (cara sesuai CLI/console yang ada —
         verifikasi dari kode; jangan asal)
       - cek `/healthz` 200 & login admin 200
5. [x] Registrasi endpoint cabang di BALI (plugin endpoint):
       - Key = jogja/semarang/lombok/java-children
       - BaseURL = `http://100.125.175.34:<port>` (Tailscale, bukan
         localhost — federation proxy jalan dari proses bali)
       - Verifikasi federation bisa meneruskan (health per endpoint).
6. [x] Daftar 3 gudang dummy per cabang (sesuai
       `~/workspaces/bbh-dummy-data/branches/<key>.yaml`):
       via UI admin atau `POST /api/admin/wh/warehouses` (akun local).
7. [x] (Opsional/dev-only) Import dummy cabang: folder
       `~/workspaces/bbh-dummy-data/data/<key>/2026-01-januari …`
       via `/import` masing-masing segmen, urut Jan→Des. Catat hasil
       counts per tabel.
8. [x] Verifikasi SSO (bila diinginkan user): issue portal-token di
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

## Hasil Eksekusi (24 Nov 2026)

Semua 4 cabang hidup di laptop ini, akses via port, tanpa perubahan
source code aplikasi (working tree ke-4 clone bersih; hanya file lokal
yang di-exclude via `.git/info/exclude`).

### Instance & port

| Cabang | APP_PORT | Postgres | KeyDB | DB | Compose project |
|---|---|---|---|---|---|
| jogja | 8101 | 15441 | 16381 | `bbh_jogja` | `bbh-jogja` |
| semarang | 8102 | 15442 | 16382 | `bbh_semarang` | `bbh-semarang` |
| lombok | 8103 | 15443 | 16383 | `bbh_lombok` | `bbh-lombok` |
| java-children | 8104 | 15444 | 16384 | `bbh_java-children` | `bbh-java-children` |

Health `/healthz` = **200** di 8101/8102/8103/8104. Login admin lokal
= **200** di keempat cabang. Postgres/KeyDB terpisah penuh per cabang
(tidak share), network & volume ber-prefix project.

### Reseed (clean-data --yes --reseed)

Keempat cabang: 9 baris dihapus dari 37 tabel (seed bawaan migrasi),
lalu `Referensi dipulihkan: [hsh_categories sec_categories]`.
Verifikasi: `hsh_categories=7`, `sec_categories=2` per cabang.

### Endpoint di bali (plugin endpoint)

| Key | BaseURL | Hasil probe dari bali |
|---|---|---|
| jogja | `http://100.125.175.34:8101` | ok (200) |
| semarang | `http://100.125.175.34:8102` | ok (200) |
| lombok | `http://100.125.175.34:8103` | ok (200) |
| java-children | `http://100.125.175.34:8104` | ok (200) |

`local` (bali) tetap aktif; keempat cabang terdaftar `status=active`,
`is_active=false` (aturan D-6: satu endpoint aktif per instance).

### Gudang dummy (3 per cabang, HTTP 201)

- jogja: Gudang Utama/Obat/Linen Jogja Dummy
- semarang: Gudang Utama/Obat/Linen Semarang Dummy
- lombok: Gudang Utama/Obat/Linen Lombok Dummy
- java-children: Gudang Utama/Obat/Linen Java Dummy

### Import dummy Jan→Des (opsional, dikerjakan)

12 bulan × 5 segmen × 30 baris per cabang. Hasil akhir per cabang:
**residents 360, visit_visits 360, wh_batches 360, hsh_entries 360,
sec_entries 360 — 0 rejected.** (jogja 13 batch/segmen karena 1 bulan
uji awal dijalankan terpisah; sisanya 12.)

Import `wh` berhasil karena 3 gudang dummy sudah terdaftar lebih dulu —
kalau belum, importer akan fail-closed menolak baris gudang asing.

### SSO end-to-end (terverifikasi)

1. Bali `POST /api/admin/portal-token {"audience":"jogja"}` → token HS256.
2. Cabang jogja `GET /api/business/whoami` + `X-Portal-Token` →
   **403 `pending`** (auto-provision enrollment `origin=sso`).
3. Admin jogja `POST /api/admin/sso-enrollments/:id/approve` → **200**.
4. Ulang `whoami` → **200**, body `{"instance":"jogja", ...}` dengan
   permission dari grup `viewer` lokal cabang.

Membuktikan `SSO_SHARED_SECRET` cabang = bali (token diterima) dan
identitas/RBAC diresolusi LOKAL tanpa callback ke pusat.

### Temuan teknis penting

- **Blocker Compose**: `docker compose` bawaan sistem **v2.5.0** menggabung
  `ports` secara ADITIF dan tidak mendukung `!reset`/`!override` →
  `docker-compose.override.yml` tidak bisa me-remap host port yang
  di-hardcode base file (5433/16379/5050) sehingga 4 clone di 1 host
  bentrok. Solusi (disetujui owner): pasang Compose **v2.32.4** di level
  user `~/.docker/cli-plugins/docker-compose` (tanpa sudo, bisa dihapus
  kapan saja). Project bali tetap berjalan normal di versi baru.
- **Baris `endpoints` milik cabang wajib**: `branch_key` default =
  `INSTANCE_KEY` dan divalidasi ke `endpoints.key`. Migrasi hanya seed
  `local`, jadi tanpa `console endpoint add --key <key>` semua tulis/baca
  resident & visit akan **400 `ErrUnknownBranch`**. Baris ditambahkan per
  cabang (data, bukan perubahan source).
- **Server IDLE**: service `app` tidak menyalakan server Go otomatis;
  server dijalankan manual (`nohup ./bin/server`). `docker compose restart app`
  TIDAK menyalakannya kembali.
- **RAM**: `pgadmin` + `ingest-runtime` dinonaktifkan via profile `tools`
  (tidak dibutuhkan bootstrap). Image app dipakai bersama
  (`bbh-branch-app:03bf716`) karena keempat clone di commit yang sama.

## Catatan Implementasi (Dev)

- `~/workspaces/bbh-branch/jogja/.env:1-70` — `.env` per cabang dibuat
  dari nol (bukan salin bali): `INSTANCE_KEY`, `APP_PORT`, `DB_NAME=bbh_<key>`,
  `APP_KEY`/`AUTH_JWT_SECRET`/`INGEST_*` acak baru. Alasan: kredensial
  copy antar instance = satu kebocoran membuka semua cabang.
- `~/workspaces/bbh-branch/jogja/.env:48-52` — `SSO_SHARED_SECRET`
  diambil NILAI PERSIS dari `.env` bali. Alasan: verifikasi token HS256
  lintas instance; beda nilai → semua portal token ditolak. Dipisah dari
  `APP_KEY` cabang agar guard `ErrSharedSecretReused` (fail-closed, cek
  material kunci bukan string) tidak menolak start.
- `~/workspaces/bbh-branch/*/docker-compose.override.yml:1-20` — override
  memakai `!override`/`!reset` pada `ports` (postgres/keydb/pgadmin/app).
  Alasan: base compose meng-hardcode 5433/16379/5050; tanpa reset, Compose
  menggabung list → bentrok 4 instance. **Butuh Compose ≥ 2.24.**
- `~/workspaces/bbh-branch/*/docker-compose.override.yml:12-24` —
  `pgadmin`/`ingest-runtime` dipindah ke profile `tools` + `ports: !reset []`.
  Alasan: RAM laptop terbatas (11 GB, 19 container lain sudah jalan) dan
  keduanya tidak dipakai untuk bootstrap/verifikasi WO ini.
- `~/workspaces/bbh-branch/*/docker-compose.override.yml:26-32` —
  `image: bbh-branch-app:03bf716` di service `app` (bukan build lokal).
  Alasan: 4 clone di commit identik → build sekali, reuse; hemat waktu/disk.
- `~/workspaces/bbh-branch/*/.git/info/exclude:1-10` — pola `.env`,
  `docker-compose.override.yml`, `.credentials/` di-exclude di repo
  LOKAL clone. Alasan: syarat WO "tanpa perubahan source code"; memakai
  `.gitignore` repo akan mengubah file tracked.
- `plugins/endpoint/services/probe.go` → dipakai `console endpoint add`
  dan `endpoint test`: probe `/healthz`. Dipakai untuk membuktikan bali
  bisa menjangkau cabang via Tailscale (bukan localhost) — semua 200.
- `plugins/resident/services/service.go:239-253` (`resolveBranchKey`) —
  dibaca untuk memastikan kebutuhan baris `endpoints`. Tanpa baris
  ber-key `INSTANCE_KEY`: `count==0` → `ErrUnknownBranch` → HTTP 400.
  Itu sebabnya tiap cabang di-`endpoint add`.
- `plugins/maintenance/services/service.go:235-241` (`Reseed`) — dipakai
  `clean-data --reseed`; memulihkan 7 kategori hsh + 2 kategori sec
  (prasyarat importer; tanpa ini semua baris hsh/sec ditolak). Terbukti
  `hsh_categories=7`, `sec_categories=2` sesudah reseed.
- `plugins/portalsso/plugin.go:52-58` — route yang dipakai verifikasi
  SSO: `POST /admin/portal-token` (izin `endpoint.write`),
  `GET /admin/sso-enrollments` + `POST /:id/approve` (izin `group.write`),
  `GET /business/whoami` (portal identity). Membuktikan mekanisme 403
  pending → approve → 200 tanpa membaca kode cabang.
- `plugins/warehouse/handlers/master.go:260-281` +
  `plugins/warehouse/services/warehouse.go:44-58` — `POST /api/admin/wh/warehouses`
  hanya butuh `name` (unik case-insensitive → 409). Dipakai mendaftarkan
  3 gudang dummy/cabang SEBELUM impor `wh`, karena importer gudang asing
  bersifat fail-closed (O-2).
- Impor dummy: urutan Jan→Des, 5 segmen via
  `POST /api/admin/{resident,visit,wh,hsh,sec}-template/import`. Hasil
  360 baris/segmen, 0 rejected. `wh` butuh gudang terdaftar lebih dulu
  (lihat poin sebelumnya) — kalau tidak, baris jadi rejected.

## Verifikasi Ulang (cara cepat)

```bash
for p in 8101 8102 8103 8104; do curl -s -o /dev/null -w "$p=%{http_code} " http://localhost:$p/healthz; done; echo
docker ps --format '{{.Names}}' | grep '^bbh-.*-postgres-1$' | wc -l   # → 4
docker compose -f ~/workspaces/bbh-data-board/docker-compose.yml exec -T postgres \
  psql -U postgres -d bbh_data_board -tAc "select count(*) from endpoints where key in ('jogja','semarang','lombok','java-children')"  # → 4
```
