---
title: "Docker App Container — Non-Root User & Host UID/GID Mapping"
description: "Container `app` tidak lagi berjalan sebagai root: user umum `go`, umask 002, dan UID/GID disamakan dengan host user agar bind mount tetap dapat ditulis."
version: 1
status: complete
created: "2026-10-23"
updated: "2026-10-23"
tags:
    - docker
    - security
    - dockerfile
    - docker-compose
    - dev-container
---
# SPEC: Docker App Container — Non-Root User & Host UID/GID Mapping

## Ringkasan
Perbaikan Dockerfile image development `app` (BBH Data Board) agar proses di dalam container **tidak berjalan sebagai root**. Container menjalankan user umum bernama `go` dengan **umask 002**, dan **UID/GID user `go` disamakan dengan host user** sehingga file yang dibuat lewat bind mount `/src` tetap bisa ditulis/dimiliki host user.

## Keputusan Desain (Disetujui)
| # | Keputusan | Nilai |
|---|---|---|
| D-1 | Prinsip keamanan | Container `app` berjalan sebagai non-root; root hanya dipakai pada fase build image (instalasi paket/toolchain). |
| D-2 | Nama user | User umum `go` (bukan nama personal) sebagai pengguna runtime. |
| D-3 | UID/GID | Melalui build arg `HOST_UID` / `HOST_GID` (default `1000`), user `go` dibuat dengan UID/GID yang sama dengan host user sehingga bind mount `/src` tetap konsisten kepemilikan file. |
| D-4 | umask | `umask 002` diterapkan untuk sesi shell (via `/etc/profile.d/umask.sh`) dan command default container, sehingga file baru dapat ditulis grup. |
| D-5 | Kepemilikan direktori | `/src`, `/go`, dan `/home/go` dimiliki user `go`; artefak swag/`go install` tetap dapat ditulis user `go`. |
| D-6 | Compose | `docker-compose.yml` meneruskan `HOST_UID` / `HOST_GID` dari environment host ke build args image `app`. |
| D-7 | Git safe.directory | Pindah ke konfigurasi `--system` agar berlaku untuk semua user (termasuk non-root `go`). |

## Goal
1. Proses default container `app` berjalan sebagai user non-root `go`, bukan root.
2. File/direktori yang dibuat di dalam container (mis. lewat `go build ./...`, `bin/server`) dapat ditulis dan dimiliki host user berkat pencocokan UID/GID.
3. Semua file baru memiliki izin ramah grup (`umask 002`) sehingga anggota grup yang sama tetap bisa mengaksesnya.
4. Perintah development yang sudah ada (`docker compose exec app go ...`) tetap berfungsi tanpa perubahan.

### Kriteria Sukses (Acceptance Criteria)
- [AC-1] `docker compose exec app id` menampilkan `uid=1000(go) gid=1000(go)` (nilai sesuai `HOST_UID`/`HOST_GID` host) dan **bukan** `uid=0(root)`.
- [AC-2] `docker compose exec app sh -c 'umask'` menampilkan `0002`.
- [AC-3] Build image `app` sukses dengan build args default (1000/1000) maupun nilai non-default (mis. 1001/1001) tanpa error `adduser`/`addgroup`.
- [AC-4] Perintah sebelumnya tetap jalan: `go version`, `swag --version`, `go build ./...`, `go test ./...`.
- [AC-5] File yang dibuat container (mis. `bin/server`) setelah build adalah milik host user (UID/GID sama), dan proses di host dapat membaca/menulisnya.
- [AC-6] Command default tetap IDLE (`sleep infinity`), dan `--user` tidak di-hardcode di compose (mengikuti build arg).
- [AC-7] `git config` safe.directory tetap berlaku untuk user non-root (`git config --system` memuat `/src` dan `*`).

## Batasan Scope
- Hanya mengubah `Dockerfile`, `docker-compose.yml`, dan dokumentasi environment (`.env.example` bila perlu).
- Tidak mengubah source code aplikasi Go.
- Tidak melakukan push ke remote.
- Compose tetap local development; user `go` **tidak** digunakan untuk image production di luar scope ini.
- Perubahan UID/GID membutuhkan **rebuild image** (`docker compose up -d --build app`); tidak ada migrasi volume/cache khusus (volume Go module cache tetap reusable selama UID/GID konsisten).

## Rancangan Normatif

### Dockerfile
- Tambahkan `ARG HOST_UID=1000` dan `ARG HOST_GID=1000` (dideklarasikan sebelum dan di dalam stage).
- Setelah instalasi utility/toolchain: buat grup `go` dengan GID `HOST_GID` dan user `go` dengan UID `HOST_UID` (primary group `go`, shell `/bin/sh`, home `/home/go`).
- Tambahkan `/etc/profile.d/umask.sh` berisi `umask 002`.
- Pindahkan `git config` safe.directory ke `git config --system` agar berlaku untuk user non-root.
- Pastikan `WORKDIR /src` dan kepemilikan `/src` / `/go` set ke `go:go`, lalu `USER go` sebelum `CMD`.
- Command default: `sh -c "umask 002 && exec sleep infinity"`.
- swag CLI dipasang sebelum `USER go` (butuh akses `/go/bin`), lalu `/go` di-chown ke `go:go`.

### docker-compose.yml (service `app`)
- `build.args` menambahkan:
  ```yaml
  HOST_UID: "${HOST_UID:-1000}"
  HOST_GID: "${HOST_GID:-1000}"
  ```
- Environment container, volume, dan `depends_on` tidak berubah.
- Tidak menambahkan `user:` di compose (UID/GID ditentukan build-time agar konsisten dengan volume go-mod-cache).

### Verifikasi
```bash
docker compose build app          # build dengan default atau env HOST_UID/HOST_GID
# atau dengan nilai kustom:
HOST_UID=1000 HOST_GID=1000 docker compose up -d --build app
docker compose exec app id        # → uid=1000(go) gid=1000(go)
docker compose exec app sh -c 'umask'   # → 0002
docker compose exec app go version
docker compose exec app go build ./...
docker compose exec app go test ./...
ls -n bin/server                  # uid/gid = host user
```

## Catatan Teknis
- Alpine `addgroup`/`adduser` dijalankan dengan guard (`getent`/`id -u`) agar idempotent lintas environment.
- `HOST_UID`/`HOST_GID` perlu dideklarasikan ulang setelah `FROM` (scope build stage).
- Gunakan `git config --system` daripada `--global` agar konfigurasi ikut terbaca saat `exec` sebagai user `go`.