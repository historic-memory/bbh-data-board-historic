---
title: wo 01 homepage
description: "Menyajikan homepage statis di path / dari Go (plugin pages)"
status: progress
created: "2026-11-24"
updated: "2026-11-24"
---
# wo 01 homepage

## Ringkasan
Menyajikan homepage statis sederhana saat akses `https://bbh.rolldev.my.id/`
(path exact `/`) dari Go, berisi link tombol menuju SPA `/app/`.

## Latar
- `/` saat ini jatuh ke fallback `c.File(p.spaIndex)` di `plugins/pages/plugin.go`,
  tapi index SPA dibangun dengan `base: /app/` → blank/tanpa asset di `/`.
- User memilih solusi dari sisi Go (bukan static di nginx, bukan `HOMEPAGE_URL`).

## Tugas
1. [x] Tambah homepage HTML statis di plugin `pages` — prefer `go:embed`
      (`plugins/pages/homepage.html`) supaya bekerja tanpa volume tambahan di
      container. Styling inline, konten: judul "BBH Data Board" + tombol
      `href="/app/"` "Masuk ke Aplikasi".
2. [x] Di `wildcardHandler`: path exact `/` → serve homepage
      (`Content-Type: text/html; charset=utf-8`).
3. [x] Pastikan fallback default (path tak dikenal lain) TIDAK berubah
      (tetap SPA index) — hanya `/` exact yang dialihkan.
4. [x] Pastikan `/healthz`, `/api`, `/app`, `/spa`, `/assets`, `/doc.json`,
      `/swagger` tidak terpengaruh (mereka match prefix/route sebelum wildcard).
5. [x] Build & jalankan: `docker compose exec app go build ./...` lalu restart
      `go run ./cmd/server`; verifikasi `curl http://localhost:8082/`
      menampilkan homepage dan `curl http://localhost:8082/app/` tetap 200.
6. [x] Regenerasi swagger tidak diperlukan (plugin `pages` tak punya endpoint
      API baru) — catat bila ternyata perlu.

## Acceptance
- `GET /` → 200, homepage HTML dengan link ke `/app/`.
- `GET /app/` → 200 SPA (tidak regressed).
- `GET /healthz` → 200 JSON.
- Tidak ada perubahan route API.

## Catatan
- File `deploy/nginx/` DIBATALKAN — sudah dihapus. Solusi murni dari Go.
- Deploy nginx VPS tetap perlu `proxy_pass` ke `http://100.125.175.34:8082`
  (tidak tergantung isi homepage).

## Catatan Implementasi (Dev)

- `plugins/pages/homepage.html:1-64` — berkas baru: homepage statis (judul
  "BBH Data Board" + tombol `href="/app/"` "Masuk ke Aplikasi"), CSS inline di
  dalam `<style>`, tanpa aset eksternal/font/CDN, tanpa JS. Kenapa: halaman root
  harus tetap tampil walau SPA belum termuat (SPA `base:/app/` tidak bisa
  dipakai di `/`).
- `plugins/pages/plugin.go:5` — import `_ "embed"`. Kenapa: `go:embed` butuh
  impor paket embed walau hanya blank (wajib agar direktif dikompilasi).
- `plugins/pages/plugin.go:26-33` — `//go:embed homepage.html` → `var
  homepageHTML []byte`. Kenapa: HTML ikut masuk binari sehingga tidak perlu
  volume/mount tambahan di container (dist SPA tidak memuatnya).
- `plugins/pages/plugin.go:44-48` — dokumentasi routing Plugin ditambah baris
  `"/" exact → homepage statis`. Kenapa: komentar routing adalah acuan utama
  siapa pun yang membaca plugin ini (mencegah orang menambah route root lagi di
  tempat lain).
- `plugins/pages/plugin.go:120-125` — `case path == "/"` sebagai cabang PERTAMA
  switch di `wildcardHandler`, pakai `c.Data(200, "text/html; charset=utf-8",
  homepageHTML)`. Kenapa: "/" exact saja (bukan prefix) supaya
  `/`, `//`-varian lain dan path tak dikenal tetap ke fallback; Content-Type
  eksplisit karena `c.Data` tidak menebak HTML seperti `c.File`.
- `plugins/pages/plugin.go:126+` — cabang `/app`, `/api`, `/spa`, proxy
  `HOMEPAGE_URL`, dan fallback `c.File(p.spaIndex)` TIDAK disentuh. Kenapa: WO
  tugas 3/4 — mencegah regresi SPA & route API.
- Tidak ada perubahan `internal/`, `cmd/`, migrasi, maupun route API. Kenapa:
  fitur murni tampilan dan sudah berada di plugin (aturan: fitur = plugin).
- Swagger TIDAK diregenerasi. Kenapa: tidak ada endpoint API baru/berubah;
  handler wildcard tidak dianotasi swagger. (Dicek: `/doc.json` tetap 200.)
- Verifikasi container `app` (host port 8082): `go build ./...` + `go vet
  ./plugins/pages/` OK; restart perlu kill by PID karena `go run ./cmd/server`
  lama (PID 15) masih memegang :8080 sehingga binari baru gagal bind.
- Hasil curl: `/` → 200 `text/html; charset=utf-8` (memuat `href="/app/"`);
  `/app/` → 200 SPA; aset `/app/assets/index-CbzXS9gG.js` → 200; `/healthz` →
  200 `{"status":"ok"}`; `/api/tidak-ada` → 404 JSON; `/halaman-tidak-dikenal`
  → 200 index SPA (fallback utuh); `/spa` → 301 `/app/`; `/doc.json` → 200;
  `/swagger/index.html` → 200; `/assets/` → 200; `/api/admin/residents` → 401
  (route API asli tetap menang atas wildcard).
