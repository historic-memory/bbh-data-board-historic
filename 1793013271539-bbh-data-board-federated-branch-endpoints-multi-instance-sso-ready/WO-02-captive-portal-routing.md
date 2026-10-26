---
title: 'WO-02: Captive Portal & Routing — pengalihan API business ke cabang'
description: 'Dashboard admin pusat dapat "Masuk ke Cabang" (captive portal). Setelah masuk, API business dialihkan ke endpoint cabang; List Portal tetap ke pusat; Admin Profile dua versi. Proxy per request via X-Branch-Endpoint, anti-loop X-Forwarded-Instance, klasifikasi error, fallback local, audit.'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 1
---

# WO-02: Captive Portal & Routing

**SPEC acuan:** `SPEC.md` (D-1..D-15, AC-1..AC-22)
**Prasyarat:** WO-01 (endpoint registry, `is_active`, `endpoint_audit`) sudah selesai.
**Fase:** 2

## 1. Tujuan

Setelah WO-01, pusat bisa **mendaftarkan** portal cabang — tetapi belum ada jalan untuk **memakainya**. WO ini membangun:

1. **Mekanisme pemilihan portal** ("Masuk ke Cabang") di sisi server + konsumsi di SPA.
2. **Pengalihan API business** ke endpoint yang dipilih — tanpa mengubah auth.
3. **Guard** supaya pengalihan tidak liar: pemilik route eksplisit, anti-loop, fallback.

## 2. Model Perilaku (ringkas)

```
Admin (SPA) ── login ─────────────► PUSAT      (main token)
    │
    │ pilih portal "cabang-jkt"
    ▼
PUSAT ── SSO (WO-03) ─────────────► CABANG     (portal token diperoleh)
    │
    │ set is_active = cabang-jkt   (WO-01, kolom is_active)
    ▼
SPA reload
    │
    ├─ menu business ──► endpoint aktif (CABANG), Authorization: portal token
    ├─ List Portal   ──► PUSAT (selalu),       Authorization: main token
    └─ Admin Profile ──► dua versi:
                           • pusat  → PUSAT,   main token
                           • cabang → CABANG,  portal token
```

## 3. Pemilik Route (D-13)

Setiap route **wajib** punya penetapan pemilik. Tanpa penetapan → **default `local`** (pusat).

| Route / Menu | Pemilik | Token |
|---|---|---|
| `POST /api/auth/login` | `local` | — |
| `GET /api/admin/endpoints*` (List Portal) | **`local`** | `main token` |
| `POST /api/admin/endpoints/:key/use` | **`local`** | `main token` |
| Profil pusat (`GET /api/admin/profile` @pusat) | `local` | `main token` |
| Menu business (`/api/business/*`, POS, inventory, laporan) | `endpoint` | `portal token` |
| Profil cabang (route sama, instance cabang) | `endpoint` | `portal token` |
| Route tak terdaftar | **`local`** (default) | `main token` |

**Aturan mati:** **List Portal & login selalu `local`.** Kalau ikut berpindah, admin terkunci di cabang yang mungkin sedang mati — dan tidak punya jalan keluar.

**Implementasi:** tabel penetapan (di kode, bukan DB) — mis. map `route-prefix → owner`. Harus bisa dibaca (dokumentasi) dan diuji.

## 4. Dua Token (D-14/D-15)

| Token | Sumber | Dipakai oleh | Dibuang saat |
|---|---|---|---|
| `main token` | Login pusat | Route `local` | Logout total |
| `portal token` | SSO (WO-03) saat masuk portal | Route `endpoint` | **Keluar portal** |

**Aturan:**
- `main token` **tidak pernah** dikirim ke endpoint cabang.
- `portal token` **tidak pernah** dikirim ke pusat.
- Keluar portal → `portal token` **dibuang**, `is_active` kembali ke `local`, `main token` tetap.
- **Bergantung pada WO-03.** Bila WO-03 belum selesai, jalur `endpoint` memakai kredensial service (`endpoints.token_enc`, disegel) — **bukan** `portal token`. **Pilih satu jalur dengan jelas, jangan campur.**

## 5. Proxy Routing Server-side

### Header

| Header | Arah | Fungsi |
|---|---|---|
| `X-Branch-Endpoint: <key>` | masuk | Override endpoint aktif untuk request ini (menang atas `is_active`) |
| `X-Forwarded-Instance: <key>` | keluar | Menandai request berasal dari instance lain (anti-loop) |
| `X-Request-Id` | keluar | ID unik untuk audit (UUIDv7) |

### Resolusi endpoint (urutan, berhenti di yang pertama cocok)

1. Header `X-Branch-Endpoint` → bila ada **dan** status `active` → pakai.
2. Kolom `is_active` di tabel `endpoints` → bila ada.
3. Fallback `local`.

**Validasi:**
- Endpoint `suspended` → **tolak** dengan error jelas (AC-6).
- `X-Branch-Endpoint` menunjuk key tak dikenal → **tolak**, jangan diam-diam fallback ke `local` (fallback senyap menyembunyikan salah tulis).

### Anti-loop (D-12)

Bila request masuk **sudah** membawa `X-Forwarded-Instance` → **tolak `409`** dengan pesan jelas. Ini mencegah pusat → cabang → pusat berputar.

Saat meneruskan request keluar, instance **wajib** menambahkan `X-Forwarded-Instance: <key instance sendiri>`.

## 6. Klasifikasi Error & Fallback (D-10)

Kegagalan endpoint **tidak boleh** membuat instance crash. Petakan ke kelas:

| Sinyal | `error_class` | HTTP ke pemanggil |
|---|---|---|
| connection refused / DNS gagal | `unreachable` | `502` |
| timeout / context deadline | `timeout` | `504` |
| endpoint menolak token (401/403) | `unauthorized` | `502` (jangan teruskan 401 mentah — membingungkan di sisi SPA) |
| endpoint balas 5xx | `bad_gateway` | `502` |

**Fallback `local`:** hanya bila **dikonfigurasi eksplisit**. **Default: TIDAK fallback.** Alasan: fallback senyap membuat admin melihat **data pusat** padahal mengira melihat data cabang — bahaya data yang jauh lebih besar daripada sekadar error. Bila diaktifkan, response **wajib** membawa penanda (mis. header `X-Endpoint-Fallback: local`) agar SPA bisa memberi tahu pengguna.

## 7. Audit

Setiap request lintas instance (`endpoint != local`) dicatat ke `endpoint_audit`:

- `request_id`, `endpoint_key`, `method`, `path`, `status_code`, `duration_ms`, `error_class`.
- **Dilarang** mencatat: body, header `Authorization`, token apa pun, query string yang memuat rahasia (AC-7).

## 8. Perubahan Frontend (`client.ts`)

Halaman/menu **tidak berubah**. Yang berubah **hanya modul API**:

- Dua jalur: `apiLocal` (main token) dan `apiPortal` (portal token).
- Interceptor memilih token **berdasarkan pemilik route**, bukan satu token global.
- `localStorage`: `main_token`, `portal_token`, `active_endpoint`.
- Keluar portal → hapus `portal_token` + `active_endpoint`, jangan hapus `main_token`.

**Penting:** perubahan ini **terbatas pada `src/api/client.ts`** (+ pemanggilan). Kalau ternyata butuh mengubah halaman, hentikan dan laporkan — itu berarti penetapan pemilik route (bagian 3) belum benar.

## 9. Acceptance Criteria

| # | AC | Verifikasi |
|---|---|---|
| 1 | Memilih portal `cabang-jkt` menetapkan `is_active` dan request business berikutnya menuju cabang | test 2 instance |
| 2 | `X-Branch-Endpoint: cabang-jkt` mengarahkan satu request **tanpa** mengubah `is_active` | test |
| 3 | Setelah masuk portal, **List Portal tetap ke pusat** dan tetap bisa berpindah/keluar (AC-17) | test |
| 4 | **Admin Profile dua versi** menampilkan data berbeda & tidak saling merusak (AC-18) | test |
| 5 | Route tanpa penetapan → **ke pusat** (AC-19) | test |
| 6 | Request dengan `X-Forwarded-Instance` **ditolak 409** (AC-20) | test |
| 7 | `main token` tidak pernah terkirim ke cabang; `portal token` tidak pernah ke pusat (AC-21) | grep/trace |
| 8 | Keluar portal membuang `portal token`; `main token` utuh (AC-22) | test |
| 9 | Endpoint `suspended` ditolak dengan pesan jelas (AC-6) | test |
| 10 | Endpoint mati → error terklasifikasi (`unreachable`/`timeout`), proses pusat **tidak crash** (AC-9) | test |
| 11 | **Tidak ada fallback senyap**: fallback hanya bila dikonfigurasi, dan ditandai | test |
| 12 | Audit terisi untuk request lintas instance, **tanpa** body/token (AC-7) | query + grep |
| 13 | `go build ./... && go vet ./... && go test ./...` lulus | CI |

## 10. Definition of Done

- [ ] Semua AC lulus dengan bukti output nyata
- [ ] Anti-loop diuji **dua arah** (ditolak & header ditambahkan saat keluar)
- [ ] Fallback senyap diuji **negatif** (tanpa konfigurasi → tidak fallback)
- [ ] Rebuild binary eksplisit (`go build -o bin/console ./cmd/console`)
- [ ] Bagian yang belum terbukti (mis. butuh instance cabang nyata) **dinyatakan belum terbukti**
- [ ] Tidak ada push ke remote

## 11. Out of Scope

- **SSO penuh & `portal token` asli** → WO-03 (WO-02 boleh memakai kredensial service sebagai jalur sementara, dinyatakan jelas)
- Rename/tambah menu di SPA
- Sinkronisasi data antar cabang
- Registrasi/enrollment admin (WO-03)
