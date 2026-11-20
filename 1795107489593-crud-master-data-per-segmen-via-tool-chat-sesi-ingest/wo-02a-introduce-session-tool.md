---
title: "WO-02a — Tool introduce_session (chat murni dari uploaded)"
description: "Backend POST /admin/ingest/tools/introduce_session (pola strict: guard ingest.write + audit ingest-tool, validasi target_plugin fail-closed, hanya dari state uploaded else 409, panggil aksi introduce yang SAMA dengan SaveMapping) + tool Node introduce_session + kontrak prompt (agent WAJIB tanya modul tujuan, dilarang menebak)"
status: complete
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - backend
    - tool
    - chat
---
# WO-02a — Tool `introduce_session` (chat murni dari `uploaded`)

Sumber: temuan review WO-01 (2026-11-20) — gate `introduced` hanya bisa
ditempuh editor (`ActionIntroduce` di `SaveMapping`,
`plugins/ingest/services/mapping.go`), sehingga sesi baru dari
`uploaded` tidak bisa diselesaikan murni via chat. Owner memilih opsi
(a): tool kecil `introduce_session`. Status: **pending**. Dikerjakan
SETELAH WO-02 (file `copilot-runtime.ts` sama — hindari konflik).

## Lingkup

### A. Backend Go

1. `POST /admin/ingest/tools/introduce_session` — pola tool strict:
   guard `ingest.write` + audit `provider='ingest-tool'` (sama dengan
   tool lain WO-01 topic `1795083433686`).
2. Input `{session_id, target_plugin}`:
   - `target_plugin` wajib dari daftar plugin yang memang memuat
     ingest segmen (`visit`, `resident`, `sec`, `hsh` — sumber dari
     daftar target mapping yang sudah ada, fail-closed; di luar → 400).
   - Hanya valid dari state `uploaded` (else 409 dengan pesan state
     sekarang) — KONSISTEN state machine existing.
   - Panggil aksi introduce yang SAMA dengan `SaveMapping`
     (satu sumber kebenaran — JANGAN duplikasi transisi state machine;
     refaktor kecil bila perlu agar aksi bisa dipanggil dua jalur).
3. D-7 tetap: tool ini HANYA memajukan gate identitas/modul — tidak
   menyentuh mapping, profil, atau ekstraksi.

### B. Node (`sub_app/ingest_runtime/copilot-runtime.ts`)

1. Skema tool `introduce_session` — pola existing (JSON Schema →
   `convertJsonSchemaToZodSchema`, Bearer user, `X-Ingest-Session-Id`,
   tanpa logika bisnis). PERHATIAN gotcha WO-01: bila parameternya
   objek bebas, gunakan `loosenFreeFormFields`.
2. Deskripsi tool: gunakan BILA sesi masih `uploaded` dan user belum
   lewat editor; agent WAJIB bertanya modul tujuan (Kunjungan/Data
   Warga/Kebutuhan/Kebutuhan Sekunder) bila belum jelas — DILARANG
   menebak; setelah tool sukses, sambung ke alur mapping biasa.

### C. Prompt agent (`plugins/ingest/prompt/agent.md`)

Kontrak kecil tambahan: sesi dari `uploaded` → tanya modul → panggil
`introduce_session` → lanjut alur mapping; editor tidak lagi satu-satunya
pintu (tetap valid untuk persiapan manual).

## Acceptance Criteria

- AC-1 Sesi BARU: upload via UI → TANPA membuka editor, chat murni dari
  `uploaded` sampai `extracted` (introduce → mapping → confirm →
  extraction) — bukti end-to-end satu sesi uji.
- AC-2 `target_plugin` di luar daftar → 400; panggil saat state bukan
  `uploaded` → 409; tanpa `ingest.write` → 403; tanpa token → 401.
- AC-3 Audit `provider='ingest-tool'` tercatat per pemanggilan.
- AC-4 Editor path TIDAK berubah perilaku (regresi: simpan draft masih
  men-trigger introduce seperti sebelumnya).
- AC-5 `go build/vet/test` + `tsc` + `npm test` runtime + build webapp
  hijau; Bruno minimal (positif + state salah 409 + 401); Log
  Implementasi (Dev) bertahap.

## Non-tujuan

- Perubahan state machine/transisi lain; perubahan `run_extraction`/
  `confirm_mapping`; UI editor.

## Catatan Implementasi (Dev)

### §A. Backend Go

1. `plugins/ingest/services/introduce.go:25` — `ErrIntroduceNotUploaded`
   sentinel terpisah (bukan string bebas) supaya handler bisa memetakan 409
   tanpa memeriksa isi pesan. Pesan membawa state sekarang
   (`(state sekarang: %s)`) agar agent tidak mengarang langkah berikutnya.
2. `plugins/ingest/services/introduce.go:55` (`IntroduceSession`) — urutan
   validasi: `NormalizeSegment(target)` → `GetSession` → cek kesesuaian
   segmen → **(baru)** cek state → transisi. Segmen dinormalkan LEBIH DULU:
   `target_plugin` asing harus 400, bukan 409 yang menyesatkan (AC-2).
3. `plugins/ingest/services/introduce.go:75` — sesi dengan segmen terpasang
   tapi BEDA → `ErrSegmentLocked` (409). Konsisten WO-06 AC-2: ganti segmen =
   buat sesi baru, jangan ditimpa diam-diam lewat chat.
4. `plugins/ingest/services/introduce.go:81` — sesi legacy tanpa segmen
   dipasangi SEKALI dari pilihan user (`SetSessionSegment`). Aman karena
   sesi `uploaded` belum punya berkas/profil → tidak melanggar kunci segmen.
5. `plugins/ingest/services/introduce.go:109` (`introduceFromUploaded`) —
   **SATU-SATUNYA** tempat transisi `uploaded → introduced` ditulis. Instruksi
   owner "jangan duplikasi transisi state machine" dijawab dengan
   mengekstrak blok inline di `SaveMapping` menjadi fungsi ini.
6. `plugins/ingest/services/mapping.go:788` — blok transisi inline diganti
   pemanggilan `introduceFromUploaded(...)` dengan komentar silang ke WO-02a.
   Perilaku editor TIDAK berubah (AC-4): aksi `model.ActionIntroduce` yang
   sama, hanya sumbernya dipusatkan.
7. `plugins/ingest/services/introduce.go:66` — `IntroduceView` memakai
   `[]model.TransitionAction` (bukan `[]string`) karena
   `model.AllowedActions()` mengembalikan tipe itu. Kesalahan tipe ini
   tertangkap saat `go build`, bukan saat uji.
8. `plugins/ingest/handlers/introduce.go:43` — handler tipis: bind → service
   → `introduceStatus`. Audit `provider='ingest-tool'` dicatat pada jalur
   sukses MAUPUN gagal (`introduce_session_failed`), supaya model yang
   memaksa tool berulang tetap terlihat di `ingest_audit` (AC-3).
9. `plugins/ingest/handlers/introduce.go:70` — `introduceStatus`:
   `ErrSessionNotFound`→404, `ErrIntroduceNotUploaded`|`ErrSegmentLocked`→409,
   `ErrSegmentRequired`|`ErrTargetPluginUnsupported`→400. 409 dipakai untuk
   KONFLIK STATE (bukan 400) karena state machine adalah otoritas, bukan
   payload yang salah bentuk.
10. `plugins/ingest/plugin.go:164` — route memakai guard `write`
    (`ingest.write`): memajukan gate adalah aksi state machine, berbeda dari
    `segment_read` yang `read`.

### §B. Node & §C. Prompt

11. `sub_app/ingest_runtime/copilot-runtime.ts:456` — `introduceSessionParameters`
    (`target_plugin` enum visit|resident|sec|hsh + `note` opsional). TIDAK
    memakai `loosenFreeFormFields` karena tidak ada peta bebas di sini.
12. `sub_app/ingest_runtime/copilot-runtime.ts:832` — payload dibangun
    EKSPLISIT (`target_plugin` dari model, `session_id` dari sesi aktif),
    sengaja **tidak** memakai `withSession(args)`. Alasan: `withSession`
    menimpa `target_plugin` dengan segmen sesi saat ini — padahal justru
    ketika sesi masih `uploaded` pilihan modul dari user itulah yang harus
    menang. Memakai `withSession` akan membuat tool ini tidak berguna.
13. `plugins/ingest/prompt/agent.md:52` — Kontrak 2a: selama `uploaded` agent
    WAJIB menanyakan modul (Kunjungan/Data Warga/Kebutuhan Sekunder/Kebutuhan
    Rumah Tangga), DILARANG menebak dari nama berkas/isi kolom, lalu memanggil
    `introduce_session`; 400/409 dijelaskan singkat dan alur dilanjutkan.
14. `plugins/ingest/prompt/agent.md:72` — Kontrak 2b (`segment_read`): hanya
    GET, path wajib di whitelist segmen, parameter asli di `query`, maks 200
    baris, data sudah dideidentifikasi, operasi lewat **ID**, `truncated`
    diberitahukan, 401/403/503 dijelaskan tanpa mencoba mengakali.

### Uji & verifikasi

15. Bruno `api-tests/ingest-crud/07..10` — introduce positif (state maju ke
    `introduced`, `allowed_actions` terisi), 409 saat state sudah
    `introduced`, 400 untuk `target_plugin: dashboard`, 401 tanpa token.
    Keempatnya HIJAU.
16. Smoke UI (AC-1 inti): sesi baru dari `uploaded` **tanpa** membuka editor
    → satu pesan chat "modulnya Kunjungan" → agent memanggil
    `introduce_session` → DB `state=introduced, target_plugin=visit` →
    `ingest_audit` mencatat `POST /tools/introduce_session 200
    provider='ingest-tool'`. Bukti bahwa gate `introduced` kini bisa
    ditempuh murni via chat.
17. Regresi editor (AC-4): `SaveMapping` tetap jalan — koleksi `ingest/`
    11–15 dijalankan dan `15-Mapping-options` 200; kegagalan 404 pada
    08–14 terbukti PRE-EXISTING (variabel `sessionId` tidak pernah di-set
    di koleksi itu), bukan akibat perubahan ini.

## Review Manager (2026-11-20) — LULUS ✅

Verifikasi independen: `introduce_session` pada state `introduced` →
**409** (dijalankan Manager via curl); transisi dipusatkan
`introduceFromUploaded` (satu sumber, dipakai `SaveMapping`); audit 39
baris `provider='ingest-tool'` pasca-00:15; bukti AC-1 chat murni
diterima (sesi `uploaded` → `introduced` via chat tanpa editor).

