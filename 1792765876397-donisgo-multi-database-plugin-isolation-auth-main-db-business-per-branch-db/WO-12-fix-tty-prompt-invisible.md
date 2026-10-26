---
title: 'WO-12: Fix Prompt Password Tidak Terlihat — /dev/tty dibuka read-only'
description: 'Bug: label "Database password (no echo):" tidak pernah muncul karena os.Open("/dev/tty") membuka file read-only (O_RDONLY), sehingga fmt.Fprint ke fd tersebut gagal EBADF dan error-nya diabaikan. Input tetap diterima karena term.ReadPassword hanya membaca. Fix: buka O_RDWR, tangani error tulis, dan tambah regression test.'
status: complete
created: "2026-10-26"
updated: "2026-10-26"
revision: 1
---

# WO-12: Fix Prompt Password Tidak Terlihat (`/dev/tty` read-only)

## Gejala (dilaporkan user)
```bash
docker compose exec -it app ./bin/console dbms add \
  --key srv-test --host postgres --port 5432 \
  --user postgres --sslmode disable --label "Server Test"
```
- **Tidak ada teks** `Database password (no echo):`
- Tapi **bisa mengetik** password (input diterima, tidak echo)
- User mengira hang → `Ctrl+C` (exit 130)

## Akar masalah (terverifikasi)
```go
// plugins/business_generator/commands.go:502
var ttyOpen = func() (*os.File, error) { return os.Open("/dev/tty") }
                                                 ^^^^^^^^^^^^^^^^^^
```
`os.Open()` = **`O_RDONLY`** (Go stdlib: "Open opens the named file for reading").

Akibatnya berantai:

| Baris | Aksi | Hasil |
|---|---|---|
| `fmt.Fprint(tty, "Database password (no echo): ")` | tulis ke fd read-only | ❌ **EBADF** — error **tidak dicek** |
| `term.ReadPassword(int(tty.Fd()))` | baca dari fd read-only | ✅ berhasil |
| `fmt.Fprintln(tty)` | tulis newline | ❌ EBADF — error tidak dicek |

Bukti reproduksi di container:
```bash
docker compose exec -T app sh -c 'exec 3</dev/null; echo test >&3'
# → sh: write error: Bad file descriptor
```

`sudo` membuka tty dengan **`O_RDWR`** — itulah sebabnya label sudo terlihat.

## Task

### 1. Buka `/dev/tty` dengan `O_RDWR`
```go
var ttyOpen = func() (*os.File, error) {
	return os.OpenFile("/dev/tty", os.O_RDWR, 0)
}
```

### 2. Jangan abaikan error tulis prompt
Error tulis **harus** terlihat — supaya bug kelas ini tidak pernah lagi gagal senyap.

Pisahkan penulisan prompt ke helper yang bisa diuji:
```go
// writePrompt menulis label prompt ke w. Error dikembalikan, bukan diabaikan.
func writePrompt(w io.Writer, label string) error {
	if _, err := fmt.Fprint(w, label); err != nil {
		return fmt.Errorf("write password prompt: %w", err)
	}
	return nil
}
```
Lalu di `readPassword`:
```go
if err := writePrompt(tty, "Database password (no echo): "); err != nil {
	return "", err   // atau fallback: tulis ke os.Stderr agar user tetap melihat label
}
b, err := term.ReadPassword(int(tty.Fd()))
fmt.Fprintln(tty)   // error boleh diabaikan (kosmetik, baris baru)
```

**Opsional tapi disarankan:** bila penulisan ke tty gagal, fallback menulis label ke `os.Stderr` — user tetap melihat petunjuk daripada buta.

### 3. Verifikasi jalur fallback
Saat `/dev/tty` tidak ada dan `os.Stdin` adalah terminal, prompt ditulis ke `tty = os.Stdin`. Pastikan perilakunya tetap benar (fd 0 pada terminal biasanya RDWR). Bila perlu, tulis label ke `os.Stderr` di jalur ini agar tidak bergantung pada mode fd stdin.

### 4. Regression test (wajib)
Tambah ke `plugins/business_generator/commands_test.go`:
- `writePrompt` ke file yang dibuka **`O_RDWR`** → sukses, isi file memuat label.
- `writePrompt` ke file yang dibuka **`O_RDONLY`** → **mengembalikan error** (uji regresi langsung untuk bug EBADF ini).
- Test lama (stdin/CRLF/empty/no-TTY) tetap lulus.

### 5. Dokumentasi
- `README.md`: contoh prompt sekarang menampilkan label; tidak perlu ubah banyak, cukup pastikan screenshot/teks contoh konsisten.

## Acceptance
- [ ] `go build ./...` + `go vet ./...` + `go test ./...` lulus (termasuk 2 test baru `writePrompt`).
- [ ] **Prompt terlihat:**
  ```bash
  docker compose exec -it app ./bin/console dbms add --key srv-x --host postgres --user postgres --sslmode disable
  # → "Database password (no echo): " TAMPIL, lalu input tidak echo
  ```
- [ ] Password yang diketik benar-benar tersimpan: `dbms list` menampilkan `srv-x`, dan `dbms_secrets.password_enc` bertambah.
- [ ] Pipe tetap diabaikan (perilaku WO-11 tidak regresi): `echo 'salah' | docker compose exec -it app ./bin/console dbms add ...` → label tampil, `'salah'` tidak dipakai.
- [ ] Non-TTY tetap error actionable + exit non-zero.
- [ ] `--password-stdin` tidak berubah.
- [ ] `grep -rn 'os.Open("/dev/tty")' --include=*.go` → **0 hit**.

## File Impact
| File | Aksi |
|---|---|
| `plugins/business_generator/commands.go` | `ttyOpen` → `O_RDWR`; helper `writePrompt`; error tulis tidak diabaikan |
| `plugins/business_generator/commands_test.go` | Tambah 2 test regresi `writePrompt` |
| `README.md` | Opsional: konsistensi contoh |

## Dependencies
- WO-11 (prompt `/dev/tty`) — WO ini **memperbaiki** implementasinya.

## Catatan
- Satu baris penyebab, tapi dampaknya besar: user buta saat mengetik rahasia dan mengira perintah hang.
- Akar kelas bug ini adalah **error yang diabaikan** pada I/O prompt. Menangani error + test regresi mencegah kambuh.
- Tidak mengubah kontrak CLI maupun alur `--password-stdin`.

## Urutan Commit (disarankan)
1. `fix(generator): open /dev/tty read-write so password prompt is visible`
2. `test(generator): writePrompt regression (RDWR ok, RDONLY errors)`
