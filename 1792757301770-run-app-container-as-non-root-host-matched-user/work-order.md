---
title: WO-01 — Non-Root Container User & Host UID/GID Mapping
description: Menjalankan container `app` sebagai user umum `go` dengan umask 002 dan UID/GID yang disamakan dengan host user agar bind mount tetap kompatibel.
status: complete
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - work-order
    - docker
    - security
    - dockerfile
---
# WO-01 — Non-Root Container User & Host UID/GID Mapping

**Referensi SPEC:** `.historic/1792757301770-run-app-container-as-non-root-host-matched-user/SPEC.md` (v1)

## Tujuan Work Order
Mengubah Dockerfile image development `app` agar tidak lagi berjalan sebagai root: buat user umum `go`, terapkan `umask 002`, dan samakan UID/GID user `go` dengan host user melalui build arg. Lalu teruskan build args tersebut dari `docker-compose.yml`.

**Checkpoint adalah goal:** setiap checkpoint dinyatakan selesai hanya jika semua kriteria penerimaannya terpenuhi dan bukti verifikasi dicatat. Kerjakan berurutan; checkpoint berikutnya bergantung pada checkpoint sebelumnya.

## Batasan
- Hanya mengubah `Dockerfile`, `docker-compose.yml`, dan dokumentasi environment bila perlu.
- Tidak mengubah source code aplikasi Go.
- Tidak melakukan push ke remote repository.
- Compose tetap local development; user `go` tidak dipakai untuk production image di luar scope.
- Build arg `HOST_UID` / `HOST_GID` didefasultkan ke `1000` agar kompatibel dengan host user standar.
- `USER go` harus disetelah seluruh instalasi root build selesai (apk, go install swag).
- Command default tetap IDLE (`sleep infinity`); tidak ada perubahan perilaku service.

## Checkpoint Goals

### CP-1 — Dockerfile: User `go` Non-Root + umask 002
**Goal:** Dockerfile membuat user `go` dengan UID/GID dari build arg, menerapkan umask 002, dan berakhir dengan `USER go`.

**Deliverable:** `Dockerfile` diperbarui.

**Kriteria penerimaan:**
- `ARG HOST_UID=1000` dan `ARG HOST_GID=1000` dideklarasikan sebelum dan sesudah `FROM`.
- Grup `go` dibuat dengan GID `HOST_GID`; user `go` dibuat dengan UID `HOST_UID` (guard idempotent).
- `/etc/profile.d/umask.sh` dibuat berisi `umask 002`.
- `git config --system` dipakai (bukan `--global`) untuk safe.directory `/src` dan `*`.
- `/src` dan `/go` di-chown ke `go:go`; `USER go` diset sebelum `CMD`.
- `CMD` default: `sh -c "umask 002 && exec sleep infinity"`.

**Verifikasi:** build image dan jalankan perintah di dalam container.

**Bukti:** hasil `id` menampilkan `uid=1000(go) gid=1000(go)` dan `umask` menampilkan `0002`.

### CP-2 — Compose: Teruskan HOST_UID/HOST_GID ke Build Args
**Goal:** `docker-compose.yml` meneruskan UID/GID host ke build args image `app` sehingga mapping dapat disesuaikan tanpa mengedit Dockerfile.

**Deliverable:** `docker-compose.yml` service `app` `build.args` diperbarui.

**Kriteria penerimaan:**
- `build.args` menambahkan `HOST_UID` dan `HOST_GID` dengan default `1000`.
- Environment container, volumes, dan `depends_on` tidak berubah.
- Tidak ada `user:` hard-coded di service `app`.

**Verifikasi:** jalankan `HOST_UID=1000 HOST_GID=1000 docker compose build app` dan periksa `docker compose exec app id`.

**Bukti:** `id` menampilkan `uid=1000(go)`.

### CP-3 — Regresi Perintah Development
**Goal:** semua perintah `docker compose exec app go ...` yang sebelumnya berfungsi tetap berfungsi sebagai user non-root.

**Deliverable:** bukti regresi.

**Kriteria penerimaan:**
- `go version`, `swag --version` sukses.
- `go build ./...` dan `go test ./...` sukses tanpa error permission.
- `bin/server` (atau artefak build lain) dimiliki host user (`ls -n` menunjukkan UID/GID = host).
- `git config --system --get-all safe.directory` memuat `/src`.

**Verifikasi:** jalankan perintah regresi melalui `docker compose exec app`.

**Bukti:** output sukses seluruh perintah dan `ls -n bin/server` menunjukkan UID/GID host.

## Bukti Verifikasi (2026-10-23)

| CP | Hasil | Bukti |
|---|---|---|
| CP-1 | ✅ selesai | `docker compose exec app id` → `uid=1000(go) gid=1000(go) groups=1000(go)`; `grep Umask /proc/1/status` → `Umask: 0002`; `git config --system --get-all safe.directory` → `/src`, `*`, `/go` |
| CP-2 | ✅ selesai | `docker-compose.yml` service `app` `build.args` memuat `HOST_UID: "${HOST_UID:-1000}"` dan `HOST_GID: "${HOST_GID:-1000}"`; image dibangun sukses (`docker compose up -d --build app`), tanpa `user:` hard-coded |
| CP-3 | ✅ selesai | `go version go1.27.3`, `swag version v1.16.4`; `go build ./...` BUILD_OK; `go test ./...` ok (pages, plugin_registry); artefak `ls -n bin/server` & `bin/console` → UID/GID `1000 1000` (host-matched) |

### Catatan implementasi
- Compose sebelumnya meng-override `CMD` Dockerfile dengan `command: ["sleep", "infinity"]` sehingga umask 002 tidak aktif di PID 1 → diganti `command: ["sh", "-c", "umask 002 && exec sleep infinity"]`.
- Volume `go-mod-cache` dan artefak lama (`bin/`) masih milik root dari container lama → diperbaiki sekali via `docker compose exec -u root app chown -R go:go /go/pkg/mod` dan `chown -R go:go /src/bin`. Untuk host lain dengan cache lama, jalankan chown serupa setelah rebuild.

## Pemetaan Checkpoint ke SPEC

| SPEC AC | Checkpoint |
|---|---|
| AC-1 — `id` = uid 1000(go) non-root | CP-1, CP-2 |
| AC-2 — `umask` = 0002 | CP-1 |
| AC-3 — build sukses dengan UID/GID default & kustom | CP-1, CP-2 |
| AC-4 — perintah `go`/`swag` tetap jalan | CP-3 |
| AC-5 — artefak milik host user | CP-3 |
| AC-6 — command default IDLE, tanpa `user:` hard-coded | CP-1, CP-2 |
| AC-7 — `git config --system` safe.directory | CP-1, CP-3 |