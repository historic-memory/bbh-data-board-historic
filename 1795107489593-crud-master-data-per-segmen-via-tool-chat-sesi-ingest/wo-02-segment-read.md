---
title: "WO-02 — Backend segment_read (whitelist + deidentify) + skema Node"
description: "POST /admin/ingest/tools/segment_read — proxy GET-only ke API admin segmen sesi dengan whitelist prefix per segmen (fail-closed di Go), RBAC baca plugin target, deidentify tunggal sebelum response ke LLM (ID utuh), paginasi max 200; skema Node pola existing"
status: complete
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - backend
    - tool
    - crud
---
# WO-02 — Backend: `segment_read` (whitelist + deidentify) + skema Node

Sumber: SPEC `spec.md` topic `1795107489593` + desain final `notes.md`
(owner ok 2026-11-20). Prasyarat: SPEC disetujui owner. Status:
**pending**. Pola ikut tool strict WO-01 topic `1795083433686`
(`analysis_tools.go` + `handlers/analysis.go`).

## Lingkup

### A. Endpoint Go

1. `POST /admin/ingest/tools/segment_read` — input
   `{session_id, path, query?}`:
   - `path` RELATIF segmen sesi (contoh segmen visit:
     `visit/visitors`, `visit/visits`); tanpa slash awal; segmen sesi
     wajib ada (`target_plugin` kosong → 400 `ErrSegmentRequired`,
     pola T-1).
   - **GET-only**: method lain → 400 (internal hanya GET; jangan
     pernah meneruskan tulisan lewat jalur baca).
   - Ambil ID admin + permission dari token user (Bearer user), BUKAN
     service token — proxy meneruskan otorisasi user.
2. **Whitelist prefix per segmen** (map statis di Go, fail-closed —
   `ErrSegmentPathForbidden` 400 bila di luar daftar):
   - `visit` → `visit/visitors`, `visit/visits`, `visit/visits/:id`,
     `visit/visitors/:id`, `visit/visits/:id/status`
   - `resident` → `residents`, `residents/:id`, `residents/stats`,
     `facilities`, `facilities/:id`
   - `sec` → `sec/*`; `hsh` → `hsh/*`, `needs/*` (map yang sederhana
     per plugin ID).
   - **LARANGAN keras** (400, apapun segmennya): `admins`, `groups`,
     `permissions`, `audit`, `ingest/*` (termasuk tools sendiri),
     `providers`, `dashboard`, `endpoints`, `resident-template`.
     Path/segment di luar daftar → 400.
3. **RBAC baca plugin target**: pemanggil wajib punya permission baca
   segmen (`visit.read`, `resident.read`, `sec.read`, `hsh.read`) —
   tanpa itu 403 (JANGAN cukup `ingest.write` untuk baca lintas modul).
4. **Deidentify tunggal sebelum response ke LLM** (P-5): response JSON
   dari API asli dilewati jalur `DeidentifyColumns` (pola
   `safeRowValues` WO-01 topic lama) — field yang ter-mapping PII
   dimask deterministik; **ID tetap utuh** agar operasi by ID mungkin;
   fail-closed: key deidentify kosong → 503 (sama dengan safe-payload).
5. **Paginasi asli API** (W-7): `query` diteruskan (q/limit/offset/
   branch_key/…); tool memotong respons max **200 baris** per petikan
   (bila API kembalikan lebih, potong + tandai `truncated`).
6. Guard `ingest.write` (masuk daftar tools) + audit
   `provider='ingest-tool'` — audit ganda dengan endpoint asli (W-8).

### B. Node

1. Skema 1 tool `segment_read` di `copilot-runtime.ts` — pola existing
   (JSON Schema → `convertJsonSchemaToZodSchema`, Bearer user,
   `X-Ingest-Session-Id`, tanpa logika bisnis).
2. Deskripsi tool: "baca data master segmen (deidentified) — jangan
   tampilkan nilai PII mentah ke user; operasi selalu by ID".

## Acceptance Criteria

- AC-1 Baca visit visitors/visits via tool → data terdeidentify, ID
  utuh, jumlah baris ≤200 + flag `truncated` bila dipotong.
- AC-2 Whitelist: path di luar segmen (mis. sesi visit meminta
  `sec/items`, atau `admins`) → 400 dengan pesan jelas.
- AC-3 RBAC: pemanggil tanpa permission baca target → 403; tanpa
  token → 401; viewer dengan `visit.read` → 200 baca.
- AC-4 GET-only: body yang mencoba tulis via jalur baca tidak berubah
  apa pun (bukti DB/audit).
- AC-5 P-5: nilai PII tidak mentah di response tool (uji kolom
  `full_name`/`phone_raw` visit → masked); fail-closed 503 bila key
  deidentify kosong.
- AC-6 Audit `provider='ingest-tool'` tercatat per panggilan.
- AC-7 `go build/vet/test` hijau + `tsc --noEmit` Node hijau; Bruno
  (positif 4 segmen + negatif whitelist/RBAC); **Log Implementasi
  (Dev)**.

## Non-tujuan

- Tulis/eksekusi (WO-03), `MergeVisitors` (WO-04), prompt agent
  (WO-05), perubahan API admin asli (kecuali bila filter baru dipakai
  — tidak, gunakan yang ada).

## Catatan Implementasi (Dev)

### §A. Endpoint Go

1. `plugins/ingest/services/segment_read.go:72` — `segmentReadWhitelist` =
   map statis per segmen (`visit` 5 pola, `resident` 8, `sec`/`hsh` pola
   `*`). Fail-closed: segmen tanpa entri tidak bisa membaca apa pun
   (AC-2 / SPEC §3).
2. `plugins/ingest/services/segment_read.go:93` — `segmentReadBannedPrefix`
   (`admins`, `groups`, `audit`, `ingest`, `providers`, …). Sebagian sudah
   tertutup whitelist, tapi daftar ini membuat penolakannya EKSPLISIT dan
   pesannya jelas (O-22) — terutama untuk `sec/*`/`hsh/*` yang berpola `*`.
3. `plugins/ingest/services/segment_read.go:160` (`SegmentReadPath`) —
   satu tempat validasi: normalisasi path → cek larangan keras → cocokkan
   whitelist. Diekspor khusus supaya bisa diuji tanpa DB.
4. `plugins/ingest/services/segment_read.go:180` (`normalizeSegmentPath`) —
   tolak path absolut (`/api/admin/...`), `://`, `?#`, `\`, `%`, segmen
   kosong, `.`/`..`. Alasan: bentuk absolut membuka jalan memutar
   (mis. `/api/admin/admins`) dan escape URL mem-bypass pencocokan pola.
5. `plugins/ingest/services/segment_read.go:216` (`matchSegmentPath`) —
   mendukung `/*` (sub-pohon) dan `:id` (SATU segmen dinamis, menolak `*`/`?`
   di dalamnya). Ini yang membedakan `visit/visits/:id` sah vs
   `visit/visits/1/2` ditolak.
6. `plugins/ingest/services/segment_read.go:246` (`SegmentRead`) — urutan
   gerbang: sesi → segmen sesi (`ErrSegmentRequired` bila kosong) → path →
   GET-only → Bearer wajib (`SegmentUpstreamError{401}`) → query →
   fail-closed deidentify → upstream → deidentify → truncate. Urutan ini
   penting: cek deidentify SEBELUM upstream supaya payload mentah tidak
   pernah sempat lewat (P-5).
7. `plugins/ingest/services/segment_read.go:333` (`segmentReadQuery`) —
   hanya skalar/daftar skalar yang diteruskan, `limit` dijepit ≤200, nilai
   kosong/nil/bukan-skalar dibuang. Model tidak bisa menyelundupkan bentuk
   yang tidak dikenal ke API asli (W-7).
8. `plugins/ingest/services/segment_read.go:415` (`fetchSegmentRead`) —
   `GET /api/admin/<path>` dengan `Authorization` MILIK USER (bukan service
   token). Konsekuensi: endpoint asli tetap penegak RBAC + audit → audit
   ganda (W-8). 401 dipaksa ada di level service juga, bukan hanya guard.
9. `plugins/ingest/services/segment_read.go:450` (`upstreamStatusForTool`) —
   401/403/404 diteruskan apa adanya, 4xx lain → 400, 5xx → 502 (bukan 500:
   kegagalan ada di upstream, bukan di tool).
10. `plugins/ingest/services/segment_read.go:487` (`deidentifyJSON`) +
    `deidentifyNode` — walker JSON rekursif; PII dimask, **ID tidak pernah
    disentuh** (AC-5, P-5). Gotcha: nilai PII bertipe OBJEK (mis. `mother`
    hasil embed) tidak dimask mentah — diturunkan dulu supaya key di
    dalamnya dinilai sendiri; tanpa ini bentuk objeknya hancur jadi string.
11. `plugins/ingest/services/segment_read.go:591` (`segmentPIITargets`) —
    sumber utama `encrypt: true` dari `fields.yaml` (via `targetFor`), plus
    alias turunan (`visitor_name`, `visitor_phone`, `mother_name`) karena API
    admin meng-embed kolom JOIN dengan nama lain.
12. `plugins/ingest/services/segment_read.go:643` (`truncateSegmentRows`) —
    kenali amplop `{items,total}` dan array polos; `truncated` = "petikan
    memuat lebih sedikit baris daripada total". **Koreksi saat uji Bruno**:
    versi pertama hanya menandai true bila SERVER memotong, sehingga
    `total=2990, returned=200` keluar `truncated:false` — menyesatkan model
    (W-7). Sekarang `truncated = total > len(items)`.
13. `plugins/ingest/handlers/segment_read.go:55` — handler hanya memetakan
    sentinel → status HTTP; kebijakan tetap di service (D-11). Pre-check
    `<segment>.read` (`middleware.Permissions`) dilakukan SEBELUM upstream
    agar penolakan 403 jelas (AC-3), lalu endpoint asli tetap memeriksa
    sendiri.
14. `plugins/ingest/handlers/segment_read.go:106` — `segmentReadStatus`:
    `errors.As` ke `SegmentUpstreamError` dulu (status asli), baru sentinel
    → 400/404/503. Deidentify kosong sengaja 503 (fail-closed, bukan 500).
15. `plugins/ingest/plugin.go:163` — route `POST /admin/ingest/tools/segment_read`
    memakai guard **`read`** (`ingest.read`), bukan `write`. Alasan: tool
    ini hanya membaca; dengan `ingest.write` viewer ber-`visit.read` akan
    403 dan AC-3 tidak mungkin lulus (keputusan owner 2026-11-20).

### §B. Node

16. `sub_app/ingest_runtime/copilot-runtime.ts:420` — `segmentReadParameters`
    memakai `loosenFreeFormFields([...], ['query'])`. Gotcha WO-01 terulang:
    konverter mematikan `additionalProperties` → Zod `catchall: never` →
    model tidak boleh mengirim `q`/`limit`/`offset` sama sekali.
17. `sub_app/ingest_runtime/copilot-runtime.ts:820` — tool `segment_read`
    memakai `withSession(args)` (segmen sesi aktif memang penentu whitelist),
    berbeda dengan `introduce_session` di bawah.
18. `sub_app/ingest_runtime/copilot-runtime.ts:570` — `segmentLabel` dapat
    `case 'hsh'` → "Kebutuhan Rumah Tangga". Tanpa ini label mentah `hsh`
    bocor ke percakapan.

### Uji & verifikasi

19. `plugins/ingest/services/segment_read_test.go` — 13 test: whitelist
    per segmen, lintas segmen, larangan keras, higiene path, `:id`, jepitan
    `limit`, deidentify (ID utuh + objek PII), `segmentPIITargets`,
    truncation (3 bentuk termasuk "upstream sudah terpaginasi"), key kosong
    fail-closed, pemetaan status upstream.
20. `sub_app/ingest_runtime/tool-schema.test.ts:93` — regresi `query` bebas
    (`q`/`limit`/`branch_key` lolos). `npm test` 21/21.
21. Bruno `api-tests/ingest-crud/` (01–11, 12 request/25 test) — HIJAU.
    Temuan uji: assertion `truncated` pertama salah PREMIS (bukan bug tool),
    lalu justru mengungkap semantik `truncated` yang menyesatkan (poin 12).
22. Smoke UI `http://100.125.175.34:5173`: sesi baru → chat "modulnya
    Kunjungan" → agent memanggil `introduce_session` → DB `state=introduced`
    → lalu "tampilkan 3 pengunjung pakai segment_read" → agent memanggil
    `segment_read` dan menjawab tabel **tanpa** `full_name`/`phone_raw`,
    menyebut `truncated: true` + `total 2990`, dan menganjurkan operasi by ID.
    `ingest_audit provider='ingest-tool'` mencatat status 200 untuk keduanya.

## Review Manager (2026-11-20) — LULUS ✅

Verifikasi independen: P-5 dibuktikan via banding DB vs respons (ID sama:
nama asli `(Rahayu) Taksu Laboratorium Klinik` → pseudonim `Luca Miller`;
HP asli `085785112259 / 081999071713` → `0853947171` — pseudonim
deterministik format-preserving, ID utuh); `admins` 400; lintas segmen
`sec/items` 400; Bruno `ingest-crud` 17 req / 37 test PASS (dijalankan
Manager); audit 39 baris pasca-00:15. **SIMPANGAN DIRATIFIKASI**: guard
route = `ingest.read` + pre-check `<segment>.read` di handler (bukan
`ingest.write` per dok awal) — baca butuh izin baca; semantik
`truncated = total > returned` diterima.

