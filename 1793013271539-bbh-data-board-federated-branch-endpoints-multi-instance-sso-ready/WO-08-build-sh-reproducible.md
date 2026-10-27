---
title: 'WO-08: build.sh tidak efektif di host (generasi swagger gagal diam-diam) dan binari tidak reproducible antar-lingkungan'
description: 'Verifikasi Manager atas WO-07 S-5 menemukan dua celah reproducibility: (1) `bash build.sh` di host gagal men-generate swagger karena swag --parseDependency tidak bisa mem-parse stdlib pada GOROOT /snap/go/11382, tetapi kegagalan ditelan `|| echo ... continuing` sehingga docs/swagger tidak berubah tanpa peringatan; (2) go build di host (snap GOROOT) dan di container (/usr/local/go) menghasilkan binari berbeda (sha 6bd87df4 vs 667448a3) meski stamp vcs.revision sama — klaim "sha cocok" hanya bermakna dalam satu lingkungan kanonik.'
status: draft
created: "2026-10-27"
updated: "2026-10-27"
revision: 1
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
