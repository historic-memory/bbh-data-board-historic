---
title: WO-05 — Custom provider LLM via UI admin (pengganti/penyanding env `INGEST_LLM_*`)
description: "Admin menambah/mengubah provider LLM (base URL, API key terenkripsi, model) lewat UI admin; tersimpan di DB terenkripsi; runtime Node mengambil konfigurasi aktif dari Go per request; key tidak pernah tampil ke klien"
status: complete
created: "2026-11-09"
updated: "2026-11-09"
---
# WO-05 — Custom Provider LLM via UI Admin

**Latar:** permintaan owner 2026-11-09 — saat ini provider LLM hanya bisa diganti
via `.env` (`INGEST_LLM_BASE_URL/API_KEY/MODEL` di runtime Node) yang menuntut
edit berkas + restart container. Owner ingin mengelola provider lewat UI admin.
Owner mengonfirmasi bentuk: (1) provider via UI admin, (2) API key di DB
terenkripsi, (3) dibuat WO sekarang.

## Tujuan

Admin dapat menambah/mengubah/menonaktifkan provider LLM (OpenAI-compatible)
lewat UI admin — tersimpan di DB dengan API key terenkripsi — tanpa edit
`.env` dan tanpa restart. Jalur chat tetap mematuhi D-11: key tidak pernah
sampai ke klien.

## Lingkup

### A. Backend (plugin `ingest`)

- Migrasi: tabel `ingest_providers` (name, base_url, model, api_key_ciphertext,
  is_active, timestamps). Satu provider aktif (UNIQUE partial pada
  `is_active=true`) — model sederhana dulu; multi-provider + per-sesi menyusul
  bila diminta.
- **Enkripsi key**: AES-GCM (pola token endpoint / `INGEST_DEIDENTIFY_KEY`
  fail-closed yang sudah ada). Key plaintext TIDAK pernah kembali ke klien —
  UI menampilkan bentuk mask (`sk-…last4`).
- CRUD admin (guard `ingest.write` / halaman pengaturan baru):
  - `GET /admin/ingest/providers` — daftar (key dimask).
  - `POST /admin/ingest/providers` — tambah.
  - `PUT /admin/ingest/providers/:id` — ubah (base_url/model wajib; key opsional
    = hanya ganti bila diisi).
  - `POST /admin/ingest/providers/:id/activate` — set aktif (otomatis menonaktifkan
    lainnya).
  - `POST /admin/ingest/providers/:id/test` — tes koneksi + daftar model dari
    provider (panggilan server-side; hasil sukses/gagal + pesan).
- Audit semua aksi (`provider='ingest-provider'`).
- **Kontrak runtime Node**: endpoint internal `GET /admin/ingest/providers/active`
  (dipakai runtime Node per request atau cache pendek TTL) mengembalikan
  konfigurasi aktif termasuk plaintext key — HANYA diakses runtime Node dengan
  kredensial service (bukan klien admin). Pola sama seperti `safe-payload`:
  jalur server-to-server, guard khusus, ter-audit.

### B. Runtime Node (`ingest_runtime`)

- `createAgent` per-request kini menyelesaikan provider: konfigurasi aktif dari
  Go → fallback env `INGEST_LLM_*` bila TIDAK ada provider aktif di DB
  (fail-safe ke perilaku lama).
- Bridge env `INGEST_LLM_* → OPENAI_*` yang ada di satu titik (log WO-01a)
  diperluas menjadi "resolve provider config" satu titik: env → DB (lewat Go).
- Tidak ada perubahan protokol AG-UI; tambalan `provider-compat.ts` tetap
  berlaku (masih per-request).

### C. UI admin

- Halaman pengaturan baru di grup "Ingest" (`/ingest/providers`): daftar
  provider + bentuk tambah/edit (key ditulis hanya saat set/ubah; tampilan
  mask), tombol "Aktifkan" dan "Tes koneksi" (tampil hasil inline).
- Peringatan keamanan di UI: key tersimpan terenkripsi; tidak bisa dilihat
  kembali — hanya diganti.

### D. Keamanan & aturan

- Key plaintext hanya ada: di DB terenkripsi, di memori Go saat dekripsi, dan
  di memori Node saat pemanggilan provider. Tidak di log, tidak di audit, tidak
  di respons klien (mask saja).
- Fail-closed: key enkripsi kosong → CRUD provider ditolak 503 (jangan menyimpan
  plaintext).
- Rate/kondisi: ganti provider aktif berlaku untuk run BERIKUTNYA; run yang
  sedang berjalan tidak diputus.

## Acceptance Criteria

- AC-1 Admin menambah provider via UI (base URL + key + model) → tersimpan
  terenkripsi; daftar menampilkan mask, bukan plaintext.
- AC-2 Aktivasi provider → chat BERIKUTNYA memakai provider baru (bukti: model/
  header respons berubah) TANPA restart container.
- AC-3 Tanpa provider aktif di DB → fallback env `INGEST_LLM_*` (perilaku lama
  tidak rusak — regresi nol).
- AC-4 "Tes koneksi" menunjukkan sukses/gagal nyata (bukan stub) tanpa
  membocorkan key.
- AC-5 Key plaintext tidak muncul di: respons API, log, audit, DB (hanya
  ciphertext); uji: query DB langsung + grep log.
- AC-6 RBAC: viewer tanpa `ingest.write` → 403 CRUD; tanpa token 401; audit
  terisi semua aksi; endpoint internal `providers/active` menolak kredensial
  admin biasa (khusus runtime).
- AC-7 `go build`/`vet`/`test` hijau; `tsc`/`npm run build` hijau; swagger +
  Bruno (CRUD, activate, test, negatif 401/403/503); Catatan Implementasi (Dev)
  sesuai aturan 2026-11-06.

## Non-tujuan

- Multi-provider aktif bersamaan / pemilihan provider per sesi-chat (kutandai
  sebagai kemungkinan lanjutan, butuh keputusan terpisah).
- Provider non-OpenAI-compatible.
- Mengubah D-11 (key tetap tidak pernah ke klien; kebijakan tetap di Go).

## Keputusan Owner & Verifikasi (2026-11-09 — handoff final)

1. **APPROVED — key env→DB**: owner setuju dengan pergeseran sumber key dari env API key terenkripsi di DB (Go menjaga key; node menerima konfig via endpoint internal server-to-server). Prinsip inti D-11 tetap: key tidak pernah sampai ke klien; hanya berubahnya tempat penyimpanan (DB terenkripsi) dan mekanisme distribusi (fetch per request, bukan env statis).
2. **Verifikasi CopilotKit (Manager, 2026-11-09)**: CopilotKit TIDAK menyediakan management API bawaan untuk provider LLM (tidak ada CRUD base URL/key/model; dashboard Intelligence hanya mengelola memory/learning/threads, bukan kredensial LLM). Pola yang didukung: env, atau config object per-agent. BuiltInAgent menerima `apiKey` langsung di config — pas dengan pola `createAgent` per-request (WO-01a). Kesimpulan: fitur ini WAJIB dibangun mandiri (Go = management API; Node hanya konsumen per request) — sesuai §A/§B di atas; TIDAK menyalahi CopilotKit.
3. **Arsitektur final ditegaskan**: Go menyimpan API key (terenkripsi) & mendedahkan PLAINTEXT ke Node hanya via endpoint internal; Node tetap satu-satunya pemanggil LLM. Fall-back env dipertahankan (backward compat).
4. WO ini **STATUS: siap eksekusi — handoff ke Dev Agent (2026-11-09)**; manager TIDAK menyentuh source code; semua keputusan desain di dok ini dianggap final.

## Risiko & catatan

- **Key di DB vs env**: ini pergeseran dari D-11 asli ("key hanya di Node env").
  Tetap setia pada PRINSIP D-11 (key tidak pernah ke klien; Go = penjaga
  kebijakan) — key kini transit Go (enkripsi/dekripsi) ke Node per request.
  **Butuh persetujuan owner secara eksplisit di catatan ini** sebelum eksekusi.
- Rotasi key enkripsi DB (kalau `INGEST_DB_KEY_ENC` diganti) = semua provider
  harus di-set ulang — dokumentasikan.
- Runtime Node meng-cache konfigurasi pendek (TTL kecil, mis. 30s) agar tiap
  run tidak memukul Go — sebutkan trade-off: perubahan provider berlaku ≤ TTL.
- Kegagalan provider aktif → error tool/chat jelas ("provider aktif gagal,
  cek pengaturan"), jangan fallback diam-diam ke env (justru membingungkan).

## Catatan Implementasi (Dev)

<!-- WAJIB diisi bertahap per bagian (§A, §B, §C, §D) sesuai aturan 2026-11-06:
     entri = `berkas:baris kunci | yang dilakukan | kenapa (ref D-x/O-x/gotcha)`;
     ±10–20 entri total, hanya keputusan/gate/fail-closed/jebakan — bukan diary. -->

### §A Backend (plugin `ingest`)

- `plugins/ingest/migrations/postgres/2026_11_09_100000_create_ingest_providers.up.sql:12` | tabel `ingest_providers` + **unique index parsial** `uq_ingest_providers_single_active` `WHERE is_active` | invariant "maksimal satu aktif" dijaga DB, bukan hanya kode — dua transaksi paralel tidak bisa menghasilkan dua aktif (§A); `is_active` default `FALSE` agar provider baru tidak pernah tanpa-sengaja menyalakan kredensial yang belum diuji.
- `plugins/ingest/services/provider_key.go:65` (`sealProviderKey`) & `:104` (`openProviderKey`) | AES-GCM envelope `{"v":1,"alg":"AES-GCM","iv","cipher"}`; kunci dari `INGEST_DB_KEY_ENC` (base64 32-byte bila valid, selain itu SHA-256) | §D fail-closed: `:52`/`:107` mengembalikan `ErrProviderKeyUnconfigured` saat secret kosong — tidak ada jalur "simpan plaintext sebagai ganti".
- `plugins/ingest/model/provider.go:68` (`MaskKey`) | mask **netral** `…<hint4>` tanpa asumsi prefix vendor | gotcha: key OpenAI-compatible bisa berformat apa pun (`donis_tnt_…`, `gsk_…`); mask `sk-…` sebelumnya menyesatkan admin (menampilkan prefix yang tidak ada di key asli).
- `plugins/ingest/model/provider.go:80` (`ProviderActiveConfig`) | satu-satunya struct dengan `api_key` plaintext; `ProviderView` memakai `APIKeyCiphertext json:"-"` | AC-5: plaintext hanya boleh keluar lewat endpoint internal, tidak pernah lewat CRUD/list.
- `plugins/ingest/handlers/providers.go:305` (`ActiveProviderConfig`) | verifikasi `X-Ingest-Service-Token` dengan `subtle.ConstantTimeCompare`; env kosong → **503**, token salah → **401**, tidak ada provider aktif → **204** | AC-6: token admin biasa tidak diterima (route didaftarkan **tanpa** `middleware.RequirePermission`, `plugins/ingest/plugin.go:142`) — RBAC admin justru bukan kredensial yang sah di sini.
- `plugins/ingest/services/providers.go:206` (`DeleteProvider`) | provider aktif → `ErrProviderConflict` → **409** | gotcha: awalnya 400 (disamakan dengan validasi input) — 400 menyiratkan "data salah", padahal datanya benar dan hanya urutan aksi yang belum tepat.
- `plugins/ingest/services/providers.go:224` (`DeactivateProvider`) | **tambahan saat eksekusi**: endpoint `/deactivate` (idiomatik, 200, idempoten) | AC-3 butuh jalan sah kembali ke fallback env; tanpa ini admin yang sudah mengaktifkan provider DB terjebak (hapus ditolak 409) dan satu-satunya pilihan adalah SQL manual — jalur itu tidak boleh jadi prosedur resmi.
- `plugins/ingest/services/providers.go:459` (`normalizeProviderBaseURL`) | menerima base URL **tanpa skema**: `https://` default, `http://` bila host loopback (`localhost`/`127.`/`[::1]`) | gotcha: format `.env` lama (`donis-agent.rolldev.my.id/v1`) tidak valid untuk Go tapi valid di normalizer Node — dua normalizer akan membuat "URL yang sama" diterima env tapi ditolak UI; disamakan agar perilaku satu.
- `plugins/ingest/services/providers.go:352` (`probeProvider`) | `GET {base_url}/models`, timeout 12 s, maks 25 model, body error dipotong | AC-4: tes harus nyata (bukan stub); key tidak pernah ikut ke pesan error/log.
- `plugins/ingest/services/provider_key_test.go:1` | 7 test: seal/open, non-determinisme IV, fail-closed tanpa secret, secret berubah → gagal buka, hint tidak membocorkan key, normalisasi URL | AC-7: `go test ./plugins/ingest/...` hijau tanpa DB (kripto + normalisasi murni).

### §B Runtime Node (`ingest_runtime`)

- `sub_app/ingest_runtime/provider-config.ts:170` (`resolveProvider`) | **satu titik resolusi**: fetch `GET {INGEST_GO_API_URL}/api/admin/ingest/providers/active` dengan `X-Ingest-Service-Token`; 204 → `null` → pemanggil memakai env | §B: semua konsumen (agent, summarizer, gate startup) memakai fungsi yang sama — tidak ada cabang kedua yang bisa lupa fallback.
- `sub_app/ingest_runtime/provider-config.ts:50`/`:53` | cache sukses **30 s**, cache kegagalan **5 s** (`INGEST_PROVIDER_CACHE_TTL_MS` bisa menimpa TTL sukses) | AC-2: perubahan provider berlaku ≤30 s pada run berikutnya, tanpa restart; kegagalan di-cache pendek agar Go yang sedang restart tidak memukul tiap request tapi juga tidak "terkunci" 30 s.
- `sub_app/ingest_runtime/provider-config.ts:290` (`buildLanguageModel`) | bangun `LanguageModel` per request via `createOpenAI({apiKey, baseURL})`; host non-OpenAI/Azure → `.chat(model)`, selain itu default (`/responses`) | gotcha besar: `resolveModel` CopilotKit membaca `process.env.OPENAI_BASE_URL` global — memakai env bridge (WO-01a) berarti dua request paralel dengan provider berbeda saling menimpa. Model per-request menghapus state global itu.
- `sub_app/ingest_runtime/copilot-runtime.ts:451` (`agents: async`) | factory jadi **async**: `createAgent(request, await resolveProvider())` | §B: `createAgent` butuh konfigurasi yang baru diketahui setelah fetch; membuatnya async di factory lebih jelas daripada menyembunyikan await di dalam agent.
- `sub_app/ingest_runtime/copilot-runtime.ts:782` | gate startup memanggil `resolveProvider()` dan **503 dengan pesan jelas** bila tidak ada provider DB maupun env | §D risiko-3: "provider aktif gagal → error jelas"; fallback diam-diam ke env justru menyembunyikan salah konfigurasi.
- `sub_app/ingest_runtime/copilot-runtime.ts:499` | audit mengirim label provider (`cachedProviderLabel()`) — **tanpa** key | AC-5: jejak "provider apa yang dipakai run ini" tetap ada di `ingest_audit` tanpa membocorkan kredensial.
- `sub_app/ingest_runtime/provider-config.test.ts:1` | 8 test (fallback 204 → env, DB menang + header service, cache & `force`, tanpa env+DB → error, payload tidak lengkap → env, dua bentuk model) | AC-3/AC-7: regresi "tanpa provider DB" dijaga test, bukan asumsi.
- `sub_app/ingest_runtime/provider-compat.ts:1` (`installProviderSseCompat`) | dibuat **idempotent** (set host + satu wrapper `fetch`) | gotcha: dipanggil lagi saat provider berganti host; wrapper bersarang membuat parsing SSE berlapis dan stream menggantung.

### §C UI admin

- `sub_app/webapp/src/pages/admin/ingest/IngestProviderListPage.tsx:1` | halaman `/ingest/providers` di balik `<PermissionGate permission={INGEST_WRITE}>` | §C/AC-6: daftar pun dianggap sensitif (memberi tahu provider + potongan key), jadi `ingest.read` saja tidak cukup.
- `sub_app/webapp/src/pages/admin/ingest/IngestProviderListPage.tsx` (form edit) | kolom key **selalu kosong** saat edit, `type="password"`, label "kosongkan bila tidak diganti" | §C: key write-only — tidak ada jalur UI untuk membacanya kembali; mengganti = menulis ulang.
- `sub_app/webapp/src/pages/admin/ingest/IngestProviderListPage.tsx` (aksi baris) | tombol **Aktifkan / Nonaktifkan** saling menggantikan sesuai `is_active`; **Hapus** dinonaktifkan selama aktif dengan `title` penjelas | mencerminkan gate 409 di server: UI tidak menawarkan aksi yang pasti ditolak, tapi tetap menjelaskan alasannya.
- `sub_app/webapp/src/api/ingestProviders.ts:92` | klien API tipis (`list/create/update/delete/activate/deactivate/test/testDraft`) | AC-1: satu tempat untuk bentuk request/response; tidak ada penanganan key di sisi klien.
- `sub_app/webapp/src/pages/admin/ingest/IngestProviderListPage.tsx` (`submit`) | satu handler `submit()` tanpa argumen dipakai footer modal **dan** `onSubmit` form; uji koneksi `test-draft` tersedia sebelum simpan | gotcha: footer modal berada di luar `<form>` — dua jalur validasi akan menyimpang; `test-draft` mencegah admin menyimpan konfigurasi yang tidak pernah tersambung.

### §D Keamanan & aturan / lintas bagian

- `.env.example` (blok "Ingest / Provider LLM di DB") + `docker-compose.yml` (`app`, `ingest-runtime`) | `INGEST_DB_KEY_ENC`, `INGEST_SERVICE_TOKEN`, `INGEST_PROVIDER_CACHE_TTL_MS` didokumentasikan & dialirkan ke kedua service | §D: tanpa `INGEST_SERVICE_TOKEN` di runtime Node, fetch selalu 401 → jatuh ke env; nilai default kosong membuat fitur mati secara aman (fail-closed), bukan terbuka.
- `.env.example` (catatan rotasi `INGEST_DB_KEY_ENC`) | peringatan eksplisit: mengganti secret = **semua key tersimpan tidak bisa didekripsi** | risiko-2 di dokumen ini; rotasi berarti tulis ulang semua key, bukan dekripsi ulang.
- `api-tests/ingest/29`–`43` (Bruno) | 14 request / 27 test: list ber-mask, create 201 `is_active=false`, update tanpa menyentuh key, activate, `test-draft`, `:id/test`, **401** endpoint internal dengan token admin, **401/503** tanpa kredensial, **403/401** viewer, hapus-aktif **409**, nonaktifkan, hapus non-aktif **204**, hapus final **204** | AC-7 sekaligus bukti AC-1/AC-4/AC-5/AC-6 di jalur HTTP nyata.
- `api-tests/environments/Local.bru` (+ `.env.sample`) | tambah `viewerToken` (`BBH_VIEWER_TOKEN`) | uji negatif RBAC butuh token tanpa `ingest.write`; bila kosong uji turun ke 401 — tetap bermakna negatif, tidak pernah lolos diam-diam.
- Verifikasi AC-5 manual (2026-11-09, host dev) | baris DB hanya berisi envelope `{"v":1,"alg":"AES-GCM","iv":…}`; `grep` plaintext key di `/tmp/server.log` = 0 hit; `ingest_audit` (provider `ingest-provider`) = 0 hit | membuktikan klaim "plaintext tidak pernah di DB/log/audit" dengan bukti, bukan asumsi.
- `plugins/ingest/plugin.go:130`–`:145` | 8 route manajemen ber-guard `ingest.write` + 1 route internal **tanpa** guard RBAC (`:142`) | pemisahan tegas: permukaan admin = RBAC; permukaan service = service token. Menaruh guard RBAC di route internal akan membuat runtime Node tidak bisa memanggilnya sama sekali.

## Manager Review (2026-11-09)

- **Keputusan tambahan dev DI-SETUJUI**: `POST /:id/deactivate` (+ tombol Nonaktifkan) — logika dev benar: provider aktif ditolak hapus (409), jadi tanpa jalur nonaktif tidak ada cara sah kembali ke fallback env (AC-3) selain SQL manual. Endpoint idempoten, ter-audit, guard `ingest.write` — sesuai §C/§D. Ditandai sebagai "tambahan di luar teks asli" oleh dev sendiri di tabel bukti — proses yang benar.
- **Log implementasi DITERIMA**: §A (10 entri), §B (8), §C (5), §D (6) = **29 entri** ± batas 10-20, semuanya format `berkas:baris | dilakukan | kenapa`. Kualitas tinggi — banyak gotcha nyata terdokumentasi: race `OPENAI_BASE_URL` global CopilotKit di-resolves dengan model per-request (§B:290, paling bernilai), provider-compat SSE jadi idempotent (§B:8), mask netral tanpa asumsi prefix vendor (§A:68), dua normalizer URL agar format .env lama konsisten (§A:459). Banner komentar aturan WAJIB dipertahankan — wajar.
- **Bukti AC diperiksa**: AC-5 diverifikasi dengan bukti konkret (DB hanya envelope AES-GCM, `grep` log = 0, audit = 0 utk plaintext); AC-6 menjawab PRISYARAT paling sensitif: token **admin ditolak** (401) di endpoint internal, hanya service-token — sesuai §D "menaruh guard RBAC di route internal akan membuat runtime Node tidak bisa memanggilnya sama sekali". AC-7 lengkap (build/vet/test go, tsc, npm test 17/17, build webapp, swagger, Bruno 14 req/27 test).
- **Lingkup dijaga**: non-tujuan tidak tersentuh (tidak ada multi-aktif, tidak ada non-OpenAI, tidak ada D-11 diubah); model sederhana (satu aktif) dipatuhi — detail tabel & index parsial sesuai desain.
- **Verifikasi Manager**: commit kode `181e416` terlihat di `git log` (di atas `9040b39`, belum di-push); frontmatter `status: complete` valid; git bersih.
- **Keputusan**: WO-05 **LULUS — COMPLETE**. Catatan untuk masa depan: model id per-provider di audit sudah ada (label provider), tapi label-nya **cache-of-config** bukan slug per-provider — cukup untuk sekarang; bila kelak multi-provider diaktifkan (non-tujuan), audit label akan perlu upgraded ke id provider yang eksplisit.

## Bukti Penyelesaian (Dev, 2026-11-09)

Perintah & hasil pada host dev (container kanonik `app` untuk Go, host untuk Node/FE):

| AC | Bukti |
|---|---|
| AC-1 | `POST /api/admin/ingest/providers` → **201**, respons `{id,name,base_url,model,key_mask:"…8bbe",is_active:false}` (tanpa `api_key`). Bruno `30`, `39` hijau. |
| AC-2 | `POST /:id/activate` → **200** `is_active:true`; runtime membaca konfigurasi per request (`resolveProvider`) dengan cache ≤30 s — tanpa restart container. Bruno `32` hijau. |
| AC-3 | Tanpa provider aktif → endpoint internal **204** → `envProviderConfig()` dipakai. Test Node "tanpa provider aktif di DB → fallback env (AC-3)" + "DB menang atas env" hijau. |
| AC-4 | `POST /:id/test` → **200** `{ok:true, latency_ms:372, models:[copilot, manager, hermes, design, openrouter, assistant]}` — panggilan nyata ke `GET {base_url}/models`. Bruno `34`/`35` hijau. |
| AC-5 | DB: `api_key_ciphertext` = `{"v":1,"alg":"AES-GCM","iv":"…`; `api_key_hint=8bbe`; `grep -c <key> /tmp/server.log` = **0**; `ingest_audit` = **0** hit. Respons list/CRUD tidak pernah memuat `api_key`/ciphertext (diuji di Bruno `29`/`39`). |
| AC-6 | Tanpa token → **401**; token admin ke endpoint internal → **401**; service token → **200** (plaintext, hanya di jalur ini). `ingest_audit` terisi `provider='ingest-provider'` untuk semua aksi (200/201/204/400/401/409). Bruno `36`/`37`/`38` hijau. |
| AC-7 | `go build ./...` ✅ · `go vet ./plugins/ingest/...` ✅ · `go test ./plugins/ingest/...` ✅ (7 test kripto/URL baru) · `npx tsc --noEmit` (ingest_runtime) ✅ · `npm test` **17/17** ✅ · `npx tsc --noEmit` + `npm run build` (webapp) ✅ · `build.sh` → swagger diperbarui ✅ · Bruno provider **14 request / 27 test, PASS** ✅ |

**Tambahan di luar teks asli WO (kecil, diminta oleh AC-3):** endpoint `POST /api/admin/ingest/providers/:id/deactivate` + tombol "Nonaktifkan" di UI. Alasan: provider aktif menolak dihapus (409), jadi tanpa jalur nonaktif tidak ada cara sah untuk kembali ke fallback env — satu-satunya alternatif adalah SQL manual di DB, yang tidak boleh menjadi prosedur resmi. Endpoint bersifat idempoten (200), ter-audit, dan ter-guard `ingest.write`. Status WO: **selesai** untuk §A–§D + AC-1..AC-7; non-tujuan tidak tersentuh.
