---
title: 'WO-11: Prompt Password Gaya sudo (/dev/tty) untuk CLI Provisioning'
description: 'Ubah readPassword() agar membaca password dari /dev/tty (terminal pengendali) seperti sudo, bukan os.Stdin. Pipe/redirect diabaikan saat mode prompt, sehingga `echo salah | ... dbms add` tidak lagi menelan password palsu. Fallback ke os.Stdin bila /dev/tty tak tersedia (tetap butuh TTY untuk prompt).'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 1
---

# WO-11: Prompt Password Gaya `sudo` (`/dev/tty`)

## Masalah
`readPassword()` saat ini memakai `os.Stdin`:

```go
// plugins/business_generator/commands.go
func readPassword(fromStdin bool) (string, error) {
	if fromStdin {
		reader := bufio.NewReader(os.Stdin)
		line, err := reader.ReadString('\n')
		...
		return strings.TrimRight(line, "\r\n"), nil
	}
	fmt.Fprint(os.Stderr, "Database password (no echo): ")
	b, err := term.ReadPassword(int(os.Stdin.Fd()))   // <-- os.Stdin
	fmt.Fprintln(os.Stderr)
	...
}
```

Akibatnya, berbeda dari `sudo`:
1. **Pipe menelan input.** `echo 'salah' | docker compose exec -it app ./bin/console dbms add ...` mengambil `'salah'` sebagai password padahal user berniat mengetik manual di terminal. `sudo` mengabaikan pipe dan selalu prompt di `/dev/tty`.
2. **Gagal di non-TTY dengan pesan teknis.** `read password: inappropriate ioctl for device` — tidak informatif, tidak menyebut perlu TTY.

## Perilaku target (seperti `sudo`)
| Skenario | Perilaku yang diinginkan |
|---|---|
| `--password-stdin` diset | Baca dari `os.Stdin` (tetap, untuk CI/pipe eksplisit) — **tidak berubah** |
| Tanpa flag, ada TTY | Prompt dari `/dev/tty`; **stdin yang dipipa/redirect diabaikan** |
| Tanpa flag, `/dev/tty` tak bisa dibuka | Fallback ke `os.Stdin`; bila bukan TTY → error jelas berbahasa manusia |
| Password kosong saat prompt | Tolak dengan error jelas (bukan simpan string kosong senyap) |

## Syarat mutlak (batasan Docker — bukan bisa diubah kode)
Prompt **wajib** TTY dialokasikan:
```bash
docker compose exec -it app ...   # ✅ -it
docker compose exec -T app ...    # ❌ tanpa TTY → /dev/tty tak bisa dibuka
```
Didokumentasikan di README + help text.

## Task

### 1. Ubah `readPassword()` di `plugins/business_generator/commands.go`
Algoritma:
```
readPassword(fromStdin bool):
  if fromStdin:                        # CI/pipe eksplisit — tetap
      line := readLine(os.Stdin)
      return trim(line)

  # ---- seperti sudo ----
  tty, err := os.Open("/dev/tty")
  if err != nil:
      # tidak ada controlling terminal → coba stdin bila itu TTY
      if isTerminal(os.Stdin):  tty = os.Stdin
      else: return error "password prompt requires a TTY; run with `docker compose exec -it ...` or pass --password-stdin"

  defer tty.Close()
  fmt.Fprint(tty, "Database password (no echo): ")
  b, err := term.ReadPassword(int(tty.Fd()))
  fmt.Fprintln(tty)
  if err != nil: return error
  pw := string(b)
  if pw == "": return error "empty password rejected"
  return pw, nil
```

Poin implementasi:
- Prompt & no-echo **ke `tty`**, bukan `os.Stderr`/`os.Stdin` (itu inti perilaku sudo).
- `/dev/tty` tidak tersedia → fallback `os.Stdin` **hanya bila** stdin adalah terminal; kalau tidak, error actionable.
- Deteksi terminal: `golang.org/x/term.IsTerminal(int(f.Fd()))` (dependency `x/term` sudah dipakai).
- **Jangan** tambah dependency baru.

### 2. Terapkan ke semua jalur input password
Cek semua pemanggil `readPassword(...)` — `dbms add` dan command lain yang mungkin meminta password (mis. `create-branch` bila di masa depan butuh). Konsisten satu helper.

### 3. Pesan error & help
- Error non-TTY: `password prompt requires a TTY; use -it (docker compose exec -it ...) or --password-stdin`.
- Help/long text: jelaskan `-it` wajib untuk prompt, `--password-stdin` untuk pipe/CI.
- Tidak ada password di log/`String()`/error (tetap patuh SPEC D-17).

### 4. Dokumentasi
- `README.md`: contoh prompt interaktif + catatan `-it`, bandingkan dengan `--password-stdin`.
- `plugins/business_generator/commands.go`: doc comment di `readPassword`.

### 5. Test
Tambah unit test (paket `business_generator`) yang bisa dijalankan tanpa TTY:
- `--password-stdin` → membaca baris stdin apa adanya (termasuk trim `\r\n`).
- stdin kosong → error jelas, bukan password kosong.
- (Opsional, sulit di CI) perilaku `/dev/tty`: tandai sebagai integration/manual, tidak wajib otomatis.

## Acceptance
- [ ] `go build ./...` + `go vet ./...` + `go test ./...` lulus.
- [ ] `docker compose exec -it app ./bin/console dbms add --key x --host postgres --user postgres --sslmode disable` → prompt `Database password (no echo):`, input tidak tampil.
- [ ] **Pipe diabaikan:** `echo 'salah' | docker compose exec -it app ./bin/console dbms add --key x --host postgres --user postgres --sslmode disable` → prompt tetap muncul di terminal, `'salah'` **tidak** dipakai.
- [ ] `docker compose exec -T app ./bin/console dbms add ...` (non-TTY, tanpa `--password-stdin`) → error jelas yang menyebut `-it` atau `--password-stdin`; **bukan** `inappropriate ioctl for device`.
- [ ] `echo 'postgres' | docker compose exec -T app ./bin/console dbms add ... --password-stdin` → tetap bekerja (CI path tidak berubah).
- [ ] Password kosong saat prompt → error `empty password rejected`.
- [ ] Unit test untuk jalur `--password-stdin` lulus.
- [ ] Tidak ada password plaintext di log/help/error.

## File Impact
| File | Aksi |
|---|---|
| `plugins/business_generator/commands.go` | Ubah `readPassword()` + doc; pesan error; help text |
| `plugins/business_generator/*_test.go` | **baru/isi** — unit test `--password-stdin` + empty |
| `README.md` | Dokumentasi prompt `-it` vs `--password-stdin` |
| `internal/businesscli/*` (deprecated) | Opsional: samakan perilaku, atau cukup beri catatan tanpa ubah |

## Dependencies
- Tidak ada WO lain yang harus selesai dulu.
- Memakai `golang.org/x/term` (sudah ada) — **tanpa dependency baru**.

## Catatan
- Perubahan ini **tidak bisa** membuat prompt muncul tanpa TTY — itu batasan proses Docker (`/dev/tty` = `ENXIO` bila tak ada controlling terminal). Yang diperbaiki adalah: mengabaikan pipe (seperti `sudo`) + pesan error yang manusiawi.
- Relevan juga untuk `create-branch` bila ke depan meminta password.
- Bukan mengubah kontrak CLI (`--key/--branch-key/--branch-name`) — hanya cara membaca rahasia.

## Urutan Commit (disarankan)
1. `feat(generator): read password from /dev/tty like sudo`
2. `test(generator): password stdin/empty cases`
3. `docs: interactive prompt requires -it`
