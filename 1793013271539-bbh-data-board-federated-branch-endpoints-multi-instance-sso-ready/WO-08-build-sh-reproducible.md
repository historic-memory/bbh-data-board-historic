---
title: 'WO-08: build.sh tidak efektif di host (generasi swagger gagal diam-diam) dan binari tidak reproducible antar-lingkungan'
description: 'Verifikasi Manager atas WO-07 S-5 menemukan dua celah reproducibility: (1) `bash build.sh` di host gagal men-generate swagger karena swag --parseDependency tidak bisa mem-parse stdlib pada GOROOT /snap/go/11382, tetapi kegagalan ditelan `|| echo ... continuing` sehingga docs/swagger tidak berubah tanpa peringatan; (2) go build di host (snap GOROOT) dan di container (/usr/local/go) menghasilkan binari berbeda (sha 6bd87df4 vs 667448a3) meski stamp vcs.revision sama — klaim "sha cocok" hanya bermakna dalam satu lingkungan kanonik.'
status: complete
created: "2026-10-27"
updated: "2026-10-27"
revision: 2
---

# WO-08: `build.sh` efektif & reproducible

**SPEC acuan:** — (tooling, bukan fitur)
**Prasyarat:** WO-07 (S-5 sudah menyelaraskan perintah swag)

## 1. Latar Belakang

WO-07 S-5 menetapkan `--parseDependency --parseInternal` sebagai sumber kebenaran penamaan definisi swagger, dan perintah itu sudah masuk `build.sh` (commit `b6c425c`). Verifikasi Manager membuktikan perintah tersebut **benar** — di dalam container, hasilnya byte-identik (`docs.go`, `swagger.json`, `swagger.yaml`).

Masalahnya ada di **lingkungan eksekusi**, bukan di perintahnya.

## 2. Temuan

### S-6a — `bash build.sh` di host adalah no-op untuk swagger

```text
$ bash build.sh
Found swag CLI - regenerating docs...
failed to parse file /snap/go/11382/src/math/rand/v2/rand.go,
  error: method must have no type parameters
swag init returned non-zero status (continuing)
```

| Uji | Hasil |
|---|---|
| GOROOT host | `/snap/go/11382` |
| GOROOT container | `/usr/local/go` |
| `swag init ... --parseDependency --parseInternal` di host | **exit=1** |
| `swag init` tanpa flag di host | exit=0, tetapi penamaan definisi campur |
| `swag init ... --parseDependency --parseInternal` di container | exit=0, byte-identik |
| `sha256 docs/swagger/*` sebelum vs sesudah `bash build.sh` host | **tidak berubah** |

`build.sh` menulis `swag init ... || echo "swag init returned non-zero status (continuing)"`. Konsekuensinya: kegagalan **tidak terlihat**, dan dokumentasi tertinggal tanpa peringatan. Untuk skrip yang tujuannya menghasilkan ulang dokumentasi, "gagal diam-diam" sama buruknya dengan "tidak pernah jalan".

### S-6b — Binari berbeda antar-lingkungan

```text
container: 667448a34a6e7746   (vcs.revision=790acf6)
host     : 6bd87df46fe58933   (vcs.revision=790acf6)
```

`go version` keduanya `go1.27.3`, tetapi GOROOT berbeda → hasil build berbeda. Jadi klaim "sha256 `bin/` cocok dengan build HEAD" hanya bermakna bila kedua sisi dibangun di lingkungan yang sama. `bin/` untracked, jadi ini tidak merusak commit, tetapi penting supaya verifikasi di masa depan tidak mengulang kesalahan `MATCH` yang menyesatkan (bandingkan dengan pelajaran S-4 di WO-07 §8.2).

## 3. Pekerjaan

| # | Pekerjaan | File |
|---|---|---|
| 1 | Buat `build.sh` **berhenti dengan exit non-zero** bila `swag init` gagal — atau minimal cetak peringatan mencolok (`WARNING: swagger TIDAK di-regenerate`) dan kembalikan status akhir non-zero agar CI/pemanggil tahu | `build.sh` |
| 2 | Deteksi GOROOT yang tidak kompatibel (`/snap/...`) dan arahkan operator menjalankan generasi di container; atau jalankan swag lewat container bila tersedia | `build.sh` |
| 3 | Dokumentasikan **lingkungan kanonik** untuk build & verifikasi: container `app` (`/usr/local/go` + `/go/bin/swag`), bukan host snap | `README.md` atau komentar `build.sh` |
| 4 | Tambahkan cara verifikasi cepat: `sha256sum docs/swagger/*` sebelum/sesudah + `cmp` terhadap build ulang di lingkungan yang sama | `build.sh` (opsional) |

## 4. Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | `bash build.sh` di host (swag gagal) **tidak** lagi mengembalikan sukses diam-diam | echo `$?` setelah run = non-zero, dan/atau ada baris WARNING eksplisit |
| 2 | `bash build.sh` di container tetap menghasilkan 3 artefak swagger byte-identik | `cmp` terhadap baseline |
| 3 | Lingkungan kanonik tertulis di repo | baca `build.sh`/`README.md` |
| 4 | `go build ./... && go vet ./... && go test ./...` tetap lulus | gate |

## 5. Definition of Done

- [ ] Kegagalan generasi swagger tidak lagi tersembunyi
- [ ] Lingkungan kanonik build terdokumentasi di repo
- [ ] Tidak mengubah artefak `docs/swagger` yang sudah terverifikasi
- [ ] Commit dokumen diverifikasi dengan `historic git log`
- [ ] Tidak ada push ke remote

## 6. Out of Scope

- Mengubah flag swag (sudah final di WO-07 S-5)
- Mengubah `bin/` menjadi tracked (tetap gitignored)
- Menambah endpoint atau mengubah anotasi handler

## 7. Hasil Verifikasi Manager (independen, 2026-10-27)

Commit diperiksa: `4ccbd79` (`build.sh`, +120/-14) dan `df11cea` (`README.md`, +11/-1).

### 7.1 Status AC

| # | AC | Verifikasi Manager | Hasil |
|---|---|---|---|
| 1 | Host gagal → non-zero + WARNING | `bash build.sh` di host (PATH + `/home/donny/go/bin`) → `EXIT=1`; log memuat 2 blok WARNING (GOROOT snap di awal + "swagger TIDAK di-regenerate" di akhir) + `BUILD SELESAI DENGAN PERINGATAN`; `Build server completed` tetap muncul; `sha256` 3 artefak **tidak berubah** (`054e6d28` / `4e9a8b13` / `c478d9a2`) | ✅ |
| 2 | Container → exit 0 + byte-identik | `bash build.sh` di container → `EXIT=0`, `Swagger docs: tidak ada perubahan isi (byte-identik)`, `BUILD OK`; ketiga hash sama dengan baseline | ✅ |
| 3 | Lingkungan kanonik tertulis | header `build.sh` (blok "LINGKUNGAN KANONIK (WAJIB)") + `README.md` seksi *Local development* (menggantikan contoh `swag init` lama) | ✅ |
| 4 | Gate | `BUILD_OK`, `VET_OK`, 10 paket `ok`, `GOFMT_CLEAN` | ✅ |

### 7.2 Jalur tambahan yang diuji (tidak diminta eksplisit, tapi penting)

| Uji | Hasil |
|---|---|
| `bash build.sh --verify-docs` | ✅ mencetak manifest `sha256sum docs/swagger/*` (3 baris) |
| `bash build.sh --bogus` | ✅ `exit=2` + pesan argumen tidak dikenal |
| `swag` hilang dari PATH (container, `PATH` tanpa `/go/bin`) | ✅ `exit=1`, `WARNING: swag CLI tidak ditemukan di PATH`, `BUILD SELESAI DENGAN PERINGATAN` |
| `go mod tidy` mengotori working tree | ✅ tidak terjadi — `git status` bersih kecuali `?? bin/` |
| Stamp binari | ✅ `vcs.revision=df11cea` (= HEAD) setelah rebuild di container |

Catatan positif: kegagalan `swag` **tidak memblokir** build `server` (biner tetap dihasilkan) tetapi exit akhir non-zero — ini kombinasi yang tepat: operator tidak terhalang, CI tetap tahu dokumentasi tertinggal.

### 7.3 Catatan kecil (bukan blocker)

`-h|--help` mencetak `sed -n '2,30p'` dari header skrip. Setelah header bertambah panjang (blok lingkungan kanonik), bantuan bisa terpotong di tengah. Tidak memengaruhi build; cukup dicek bila header diedit lagi.

### 7.4 Status

WO-08 **`complete`** — S-6a (kegagalan tersembunyi) dan S-6b (lingkungan kanonik) tertutup dan terbukti dengan output perintah nyata. Artefak `docs/swagger` tetap byte-identik dengan yang diverifikasi di WO-07, sehingga perbaikan ini berlaku mundur tanpa membatalkan verifikasi sebelumnya.
