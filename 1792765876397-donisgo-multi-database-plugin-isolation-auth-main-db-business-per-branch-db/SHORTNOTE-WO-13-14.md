---
title: 'SHORTNOTE: Handoff WO-13 (`dbms test`) + WO-14 (`dbms update`)'
description: 'Catatan handoff ke Dev Agent: urutan eksekusi, titik integrasi kode terverifikasi, jebakan yang sudah ditemukan, dan definisi selesai per AC. Berisi 3 koreksi atas draft WO (format DSN, RotateDBMSSecret, payload Status).'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
---

# Shortnote Handoff — WO-13 & WO-14

**Untuk:** Dev Agent
**Sumber:** `WO-13-dbms-test.md`, `WO-14-dbms-update.md`
**Urutan disarankan:** **WO-14 dulu**, lalu **WO-13**.
Alasan: WO-13 membutuhkan `UpdateDBMS` hanya secara tidak langsung, sedangkan WO-14 menyentuh `dbms update` yang lebih sederhana dan menambal hint basi. WO-13 butuh probe jaringan baru (lebih berisiko), jadi dikerjakan setelah fondasinya stabil.

---

## BAGIAN A — Koreksi atas draft WO (WAJIB dibaca sebelum mulai)

Saya memverifikasi ulang kode setelah draft ditulis. **Tiga hal di WO perlu dikoreksi.** Ikuti shortnote ini bila bertentangan dengan WO.

### A-1. Format DSN: keyword/value, bukan URI

WO-13 menyebut `BuildDSN` menghasilkan URI. **Salah.** Kode aktual (`internal/db/gorm.go:42` `openGormDBForEntry`) memakai keyword/value:

```go
dsn := fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
    entry.Host, entry.Port, entry.UserName, password, entry.DatabaseName, entry.SSLMode)
```

**Wajib konsisten** dengan format ini. Jangan memperkenalkan `postgres://...` — akan membuat dua jalur DSN yang bisa berbeda perilaku (khususnya parsing `sslmode`).

### A-2. `RotateDBMSSecret` BUKAN untuk mengganti password baru

Ini jebakan paling berbahaya. `internal/db/secret.go:126`:

```go
func RotateDBMSSecret(main *gorm.DB, key string) error {
    // membaca password LAMA, decrypt, lalu SaveDBMSSecret(plain) lagi
    // keyVersion++ bila blob terenkripsi
}
```

Fungsinya **re-enkripsi password yang sama** (untuk migrasi master key), **bukan** menetapkan password baru. Kalau `--rotate-password` memakai fungsi ini, password **tidak akan berubah** — dan user akan mengira sudah berganti.

**Yang benar untuk `--rotate-password` dengan password baru:**
```go
db.SaveDBMSSecret(main, key, newPlaintext, 1)
```
`SaveDBMSSecret` melakukan UPSERT ke `dbms_secrets` dengan ciphertext baru + `rotated_at = NOW()`.

Nama `RotateDBMSSecret` menyesatkan. Jangan "memperbaiki" namanya di WO ini (di luar scope); cukup jangan salah pakai, dan tambahkan komentar di call-site.

### A-3. `PUT /api/admin/dbms/:key` saat ini **merusak** `status`

`admin_handler.go:78` memanggil `AddDBMS` (upsert penuh) dengan `payloadEntry(p)`, dan `payloadEntry` **tidak menyalin `Status`**:

```go
func payloadEntry(p dbmsPayload) *db.DBMSEntry {
    return &db.DBMSEntry{ /* ... Status tidak disalin ... */ }
}
```

Lalu `AddDBMS` punya default `if entry.Status == "" { entry.Status = "active" }`.

**Akibat:** memanggil `PUT` pada server berstatus `suspended` akan **diam-diam mengaktifkannya kembali.** Ini bug laten yang sudah ada sekarang, bukan hanya risiko WO-14. Karena WO-14 mengubah handler ini jadi partial, **perbaiki sekaligus**: tambahkan `Status` ke `dbmsPayload` dan `payloadEntry`, tetapi jangan diterapkan bila field tidak hadir (semantik partial = pointer/`json.RawMessage`).

### A-4. Masking: jangan bocorkan panjang password

Contoh output di WO-13 menulis `***-postgres` — itu membocorkan sebagian password. **Pakai `***` saja**, tanpa panjang, tanpa karakter.

---

## BAGIAN B — Titik Integrasi Terverifikasi

| Simbol | Lokasi | Dipakai untuk |
|---|---|---|
| `openGormDBForEntry(entry)` | `internal/db/gorm.go:42` | pola build DSN + ping (rujukan format) |
| `openDBMSPassword(enc)` | `internal/db/secret.go` (unexported) | dekripsi ciphertext |
| `LoadDBMSPassword(main, key)` | `internal/db/secret.go:47` | **Stage 1** (`dbms test`) — load + decrypt sekaligus |
| `SaveDBMSSecret(main, key, plain, kv)` | `internal/db/secret.go` | password baru pada `--rotate-password` |
| `RotateDBMSSecret(main, key)` | `internal/db/secret.go:126` | **JANGAN** dipakai untuk password baru (A-2) |
| `Pool.GetGorm(name)` | `internal/db/registry.go:319` | connection **ter-cache** — jangan dipakai untuk test |
| `db.GetGorm(name)` | `internal/db/registry.go:407` | shortcut pool default |
| `db.InvalidateAll()` | `internal/db/registry.go:436` | wajib setelah `dbms update` |
| `newDBMSAddCmd/ListCmd/RmCmd` | `commands.go:72/138/165` | tempat menambah `test` & `update` |
| `NewDBMSCmd()` | `commands.go:54` | daftarkan subcommand baru di sini |
| `Plugin.ConsoleCommands()` | `plugin.go:88` | sudah memanggil `NewDBMSCmd`, tidak perlu diubah |
| `AdminHandler.UpdateDBMS` | `handlers/admin_handler.go:78` | ubah jadi partial (A-3) |
| `RegisterRoutes` | `plugin.go:62` | tambah `POST /admin/dbms/test/:key` |
| `readPassword(fromStdin)` | `commands.go:507` | pakai ulang untuk `--rotate-password` |

### Catatan penting tentang `Pool.GetGorm`
Pool **meng-cache** koneksi. Kalau `dbms test` memakai `GetGorm(name)`, hasilnya bisa **stale** (pool lama dari host/port lama) dan test akan melaporkan "up" padahal server sudah pindah. Untuk pengujian, buat helper baru yang **tidak** menyentuh cache:

```go
// TestConnection membuka koneksi sekali pakai (tanpa cache) lalu menutupnya.
func TestConnection(entry *DBRegistryEntry) error
```

---

## BAGIAN C — WO-14 (`dbms update`) — Instruksi Eksekusi

**Scope file:** `internal/db/registry.go`, `plugins/business_generator/commands.go`, `services/admin_service.go`, `handlers/admin_handler.go`, `commands_test.go`.

1. **`DBMSUpdate` pakai pointer** (`*string`, `*int`) agar "tidak diberikan" ≠ "dikosongkan".
2. **Deteksi flag dengan `cmd.Flags().Changed("label")`** — bukan `label != ""`. Tanpa ini, user tidak bisa mengosongkan label.
3. **`UpdateDBMS` = read-modify-write dalam `main.Transaction`** → dapat `before` snapshot untuk diff, dan bisa menulis `NULL`.
4. **Urutan operasi (penting):**
   ```
   load before → validasi key ada → update metadata (tx) → bila rotate: SaveDBMSSecret(nilai BARU) → InvalidateAll → render diff
   ```
   `InvalidateAll` **setelah** secret baru tersimpan; kalau dibalik, pool bisa ter-rebuild dengan password lama.
5. **Key immutable** — tidak ada flag rename. Bila user mencoba, tolak dengan pesan yang menyebut alasan (FK `db_registry.dbms_key`).
6. **`--password-stdin` tanpa `--rotate-password`** → error actionable, jangan diam-diam diabaikan.
7. **Tanpa satu pun flag field** → error `nothing to update; specify at least one field flag`.
8. Sekalian perbaiki `internal/db/registry.go:147`: hint `bin/business` → **`bin/console`**.

**Diff harus dari nilai sebenarnya**, bukan dari flag. Contoh yang salah: menampilkan `port: 5432 → 5433` padahal user mengirim `--port 5432` (tidak berubah). Bandingkan `before` vs `after` hasil query.

### Definition of Done WO-14
- [ ] `dbms update --key k --label "x"` → berubah, **tanpa** prompt password
- [ ] `password_enc` **identik** sebelum/sesudah update metadata (buktikan via query)
- [ ] `--rotate-password` benar-benar mengubah `password_enc` **dan** password baru bisa dipakai konek (ini yang menguji A-2)
- [ ] `--password-stdin` tanpa `--rotate-password` → error
- [ ] Tanpa flag → error
- [ ] Key tak dikenal → error memuat `bin/console dbms add`
- [ ] Diff tampil per field yang **benar-benar** berubah
- [ ] `--status suspended` → tersimpan, **tidak** ter-reset ke `active` oleh PUT (A-3)
- [ ] Setelah update `host`/`port`, koneksi berikutnya memakai nilai baru (uji `InvalidateAll`)
- [ ] `PUT /api/admin/dbms/:key` partial; response tanpa `password`
- [ ] `go test ./plugins/business_generator/... ./internal/db/...` lulus

---

## BAGIAN D — WO-13 (`dbms test`) — Instruksi Eksekusi

**Scope file:** `internal/db/registry.go` (tipe hasil + `TestDBMS` + `TestConnection`), `commands.go` (`newDBMSTestCmd` + auto-test di add), `services/admin_service.go`, `handlers/admin_handler.go`, `commands_test.go`.

### Urutan stage (jangan diubah — ini yang membuat root cause terbaca)

```
1. secret   : LoadDBMSPassword(main, key)
2. dial     : net.DialTimeout("tcp", host:port, timeout)
3. tls      : handshake bila sslmode != "disable"
4. ping     : TestConnection(entry) ke database target
5. branches : untuk setiap db_registry WHERE dbms_key = key → TestConnection
```

**Aturan cascade:** stage gagal → stage berikutnya `skipped`, **bukan** `failed`. Perbedaan ini yang memisahkan "akar masalah" dari "gejala turunan". Tanpa itu, output jadi dinding error dan user tetap bertanya "entah kenapa".

**Jangan abort pada kegagalan.** Kumpulkan semua hasil, baru simpulkan.

### Pemetaan error → kalimat aksiabel

Bukan sekadar meneruskan `err.Error()`. Petakan minimal:

| Sinyal error | Kalimat untuk user |
|---|---|
| `connection refused` | server menolak koneksi — kemungkinan DBMS mati atau port salah |
| `no such host` / `server misbehaving` | nama host tidak resolve — periksa DNS |
| `i/o timeout` | tidak ada respons — kemungkinan firewall/port tertutup |
| `password authentication failed` | user/password salah — coba `dbms update --key <k> --rotate-password` |
| `database "…" does not exist` | database target belum dibuat — cek `db_registry` |
| `certificate signed by unknown authority` | CA tidak dikenal — periksa `sslmode` |
| `does not support SSL` | server tanpa TLS — turunkan `sslmode` bila memang aman |
| `no credentials registered` | secret belum ada — daftar ulang atau rotasi password |

### Hal teknis yang mudah salah
- **`--all` tidak boleh berhenti** di kegagalan pertama.
- **Timeout** per stage (default 5s), sediakan `--timeout`. Tanpa timeout, `dbms test` pada host mati bisa menggantung lama.
- **Stage 3 (TLS) itu aproksimasi.** `sslmode` punya gradasi (`disable`/`allow`/`prefer`/`require`/`verify-ca`/`verify-full`). Handshake probe hanya bermakna penuh untuk `require`/`verify-*`. Untuk `prefer`/`allow`, bila handshake gagal tetapi stage 4 sukses, laporkan sebagai **catatan**, bukan kegagalan. Jangan mengarang kepastian.
- **Masking** `***` saja (A-4).

### Auto-test di `dbms add`
Non-blocking: bila gagal, cetak **peringatan ke stderr** dan **tetap sukses** (data sudah tersimpan). Jangan mengubah exit code `dbms add`.

### Definition of Done WO-13
- [ ] `dbms test --key k` menampilkan stage diagram dengan status per stage
- [ ] `dbms test --all` menguji semua server, tidak berhenti di kegagalan pertama
- [ ] Kegagalan menyebut **sebab yang bisa ditindaklanjuti** (bukan hanya `down`)
- [ ] Stage setelah kegagalan = `skipped`
- [ ] Server tanpa cabang → kolom branches kosong, bukan error
- [ ] Semua cabang yang menunjuk server diuji sebagai baris terpisah
- [ ] Status gabungan: `pass` / `fail` / `degraded`
- [ ] `dbms add` mencetak warning bila test gagal, **exit code tetap 0**
- [ ] `POST /api/admin/dbms/test/:key` mengembalikan JSON, **tanpa** password
- [ ] Tidak ada password di output/log (grep hasil test)
- [ ] `go test ./plugins/business_generator/... ./internal/db/...` lulus

---

## BAGIAN E — Kontrak Output (agar bisa diverifikasi)

Tetapkan exit code eksplisit supaya bisa dipakai CI:

| Kondisi | Exit |
|---|---|
| Semua stage + semua branch OK | `0` |
| Server OK, ada branch gagal (`degraded`) | `1` |
| Ada stage gagal (`fail`) | `1` |
| Key tidak ditemukan / argumen salah | `2` |

Kalau kamu punya alasan kuat untuk membedakan `degraded` dari `fail` di CI, usulkan — jangan diam-diam memilih.

---

## BAGIAN F — Yang TIDAK boleh dilakukan

- Jangan mengubah skema/migrasi. Semua kolom sudah ada di `dbms`.
- Jangan menambah flag `--password` (dilarang sejak SPEC D-17).
- Jangan menambah rename key.
- Jangan mengubah `RotateDBMSSecret` (di luar scope — cukup jangan dipakai salah).
- Jangan memakai `Pool.GetGorm` untuk pengujian (cache → hasil menyesatkan).
- Jangan menyentuh `SPEC.md`; kalau kontrak perlu berubah, laporkan lebih dulu.

---

## BAGIAN G — Setelah selesai

1. Rebuild binary **secara eksplisit** — `bin/` tidak ikut `go build ./...`:
   ```bash
   docker compose exec -T app sh -c 'cd /src && go build -o bin/console ./cmd/console'
   ```
2. Lampirkan bukti output nyata per AC (bukan klaim "sesuai WO").
3. Bila ada bagian yang belum bisa diverifikasi (misalnya butuh TTY manusia), **katakan belum terbukti** — jangan ditandai selesai. (Pelajaran dari WO-11 → WO-12.)
