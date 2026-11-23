---
title: WO-09 — Integrasi AI chat (plugin ingest)
description: 'Segmen wh: baca stok via chat, usul transaksi, terapkan'
status: complete
created: "2026-11-23"
updated: "2026-11-23"
tags:
    - warehouse
---
# WO-09 — Integrasi AI chat (plugin `ingest`)

## Prasyarat

WO-08 lulus; keputusan O-3 owner.

## Tujuan

EX-2 owner: modul gudang bisa dibaca/diusul lewat AI chat, memakai pola topic `1795107489593` (segment tools) — STRICTNESS tetap di Go (D-11 topic ingest).

## Lingkup (sesuai O-3 final)

- **Baca** — segmen `wh` di whitelist `segment_read` (`plugins/ingest/services/segment_read.go`): endpoint GET-only (summary/batches/items) fail-closed; stok bukan PII → tanpa deidentify; paginasi + `truncated`; guard `warehouse.read` pre-check.
- **Tulis** — whitelist TULIS terpisah di `segment_write_propose` (endpoint+method eksplisit, usulan hanya `POST /api/admin/wh/batches` dan `POST /api/admin/wh/movements`): kartu usulan di chat → user setujui (`confirmed_by_user` wajib) → `apply_record_actions` eksekusi HTTP self-call Bearer USER (RBAC+validasi asli jalan; stok kurang → 409 diteruskan ke chat).
- Master kategori/produk TIDAK di chat (dibuat via UI) — mencegah agent mengarang master.
- tool-schema Node: objek bebas WAJIB `loosenFreeFormFields` (gotcha `convertJsonSchemaToZodSchema`).
- agent.md: kontrak gudang (baca→usul→setujui→terapkan; PII tidak ada; gudang/produk harus dikenal).

## Acceptance Criteria

- AC-1 Chat: "stok minyak angin di gudang X?" → jawaban dari summary (bukan karangan).
- AC-2 Chat: usul masuk/keluar → kartu konfirmasi → setujui → data nyata; tolak usulan tanpa persetujuan.
- AC-3 Endpoint di luar whitelist → 400 fail-closed + audit `ingest_audit provider='ingest-tool'`.
- AC-4 Bruno `ingest-crud` diperluas segmen wh; regresi nol koleksi existing.

## Non-tujuan

Tool edit master, tool impor excel via chat (bisa jadi WO lanjutan bila owner mau).

## Review Manager (2026-11-23) — LULUS ✅ (WO TERAKHIR)

Verifikasi independen Manager:

| Cek | Hasil |
|---|---|
| `git show --stat 02e7661` | 36 berkas +1451/−17; scope: `plugins/ingest/**` + UI ingest + Bruno/README — **AdminLayout/index.css tak disentuh** ✅ |
| `go test ./plugins/ingest/... -count=1` (container, diulang Manager) | 4 paket ok ✅ |
| Spot-check `record_write.go` | Whitelist tulis HANYA `wh/batches` + `wh/movements` (baris 113-114) — persis O-3 ✅ |
| **Bukti E2E dari tab browser sesi uji** (`/app/ingest/01a20893-…/chat`) | Kartu usulan **"1 aksi · applied / 1 berhasil"** (create → `wh/movements`); jawaban stok DARI SERVER (`total sisa 48`, batch id, qty_in 48); alur baca → usul ("Belum ada yang ditulis — ini baru usulan") → setujui → terapkan ✅ |
| DB | 7 tabel `wh_*` = **0 semua**; sesi uji dihapus ✅ |
| Proses | 11 commit lokal tanpa push; log implementasi; master index WO-09 complete + baris bukti ✅ |

**Catatan review:**
1. Bug prompt `moved_at` RFC3339 tertangkap lewat E2E nyata (bukan unit test) — diperbaiki di agent.md; pelajaran bagus.
2. Keputusan segmen chat-only (`supported:false`, upload 400, mapping disembunyikan) konsisten O-3 — master tetap via UI.
3. Catatan dev "di luar lingkup" (`seg.supported` di mapping picker, `target_plugin` tidak menambah warehouse) — disetujui Manager (diff minimal, tanpa jalur mati baru).

**WO-09 DITERIMA. SEMUA WO-01..09 COMPLETE — Manager menutup topik: `spec.md` → complete, `work-order.md` → complete, `notes.md` → complete (lihat section Penutup Topik di master index).**

Jabatan: Manager Agent — Nama Tugas: Review WO-09 & penutupan topik. Sayonara.

## Catatan Implementasi (Dev)

### 1. Segmen chat-only (keputusan desain inti)

O-3 menetapkan modul gudang dilayani **chat saja** (tanpa berkas/mapping). Katalog
segmen sebelumnya memaksa `Supported=true` (harus punya `TargetFields`) — gudang
tidak punya `schema/fields.yaml`, jadi ia akan selalu "belum tersedia" dan sesi
tidak bisa dibuat. Solusi minimal: tambah penanda eksplisit.

- `plugins/ingest/services/segments.go:37` — field `ChatOnly bool json:"chat_only"`
  (aditif; `Supported` tidak diakali, tetap `false` untuk gudang).
- `plugins/ingest/services/segments.go:59` — entri katalog
  `{Plugin:"warehouse", Label:"Gudang", ChatOnly:true, ...}`.
- `plugins/ingest/services/segments.go:84` — `IsChatOnlySegment()` dipakai jalur
  unggah & UI; `Segments()` tetap menghitung `Supported` dari `targetFieldsRaw`,
  jadi `TargetFields("warehouse")` tetap `ErrTargetPluginUnsupported` (fail-closed
  untuk endpoint mapping/ekstraksi).
- `plugins/ingest/services/segments.go:100` — `NormalizeSegment` diubah: validasi
  **keanggotaan katalog**, bukan keberadaan katalog field. Tanpa ini gudang tidak
  bisa dipilih saat `POST /sessions` (400 dari tempat yang salah).

### 2. Whitelist baca (fail-closed, tanpa deidentify)

- `plugins/ingest/services/segment_read.go:101-107` — 7 path relatif:
  `wh/summary`, `wh/batches`, `wh/batches/:id`, `wh/batches/:id/movements`,
  `wh/items`, `wh/categories`, `wh/warehouses`. Semua GET-only; `:id` = satu
  segmen dinamis. Tidak ada wildcard `wh/*` — endpoint impor/template sengaja
  tidak terjangkau.
- Tanpa entri di `segmentPIITargets` → cabang deidentify **dilewati** karena
  `len(pii)==0`; ini bukan pelonggaran, melainkan konsekuensi stok/bahan bukan
  PII. Banned-prefix (`admins`, `audit`, `ingest`, `*-template`, …) tetap berlaku
  global.

### 3. Whitelist tulis (terpisah dari baca)

- `plugins/ingest/services/record_write.go:113-114` — hanya dua pasangan
  pattern+method: `POST wh/batches` (batch masuk) dan `POST wh/movements`
  (keluar/penyesuaian). **Tidak ada** PUT/PATCH/DELETE sama sekali: gudang
  mencatat pergerakan, bukan mengubah riwayat. Master (`wh/categories`,
  `wh/items`, `wh/warehouses`) juga tidak ada → agent tidak bisa mengarang master.
- Konsekuensi yang diterima: koreksi salah input lewat chat = usul movement
  penyeimbang (`ADJUST`), bukan edit batch. Ini konsisten dengan D-3
  (stok = agregasi, riwayat tak diubah).

### 4. Guard unggah berkas untuk segmen chat-only

- `plugins/ingest/services/uploads.go:136` — setelah sesi & segmen dinormalkan,
  `IsChatOnlySegment` menolak **400** dengan pesan
  `"segmen warehouse dilayani lewat chat saja (tanpa unggah berkas) …"`.
  Ditaruh sebelum cek state agar tidak pernah ada sesi gudang ber-berkas yang
  lalu masuk alur mapping yang tidak ada (state setengah jalan).

### 5. UI jujur (tanpa janji palsu)

- `sub_app/webapp/src/api/ingest.ts:93,97,105` — tipe `chat_only?`,
  `selectableSegment()` (`supported || chat_only`), `isChatOnlySession()`.
- `sub_app/webapp/src/pages/admin/ingest/IngestSessionListPage.tsx:87,605` —
  picker segmen memakai `selectableSegment`; tautan mapping disembunyikan untuk
  sesi chat-only.
- `sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx:481,505,540,545` —
  flag `chatOnly` menyembunyikan "Buka editor mapping →", `<IngestUploadPanel>`,
  dan `<RejectedListingCard>`; kartu usulan (`RecordActionsCard`) tetap tampil
  karena itu jalur tulisnya.
- `sub_app/webapp/src/layouts/AdminLayout.tsx` & `src/styles/index.css`
  **tidak disentuh** (perubahan milik workstream lain di working tree).

### 6. Tool-schema Node

- `sub_app/ingest_runtime/copilot-runtime.ts:428` — deskripsi parameter `path`
  pada `segment_read` menyebut contoh gudang (`wh/summary`, `wh/batches`, …).
- `sub_app/ingest_runtime/copilot-runtime.ts:735` — `segmentLabel()` kasus
  `warehouse` → `"Gudang"`.
- Enum `target_plugin` pada `introduce_session` sengaja **tetap**
  `visit|resident|sec|hsh`: segmen chat-only tidak pernah melewati gate itu
  (lihat Kontrak 2a), jadi menambah `warehouse` justru membuka jalur yang salah.

### 7. Kontrak prompt

- `plugins/ingest/prompt/agent.md:71` — pengecualian chat-only di Kontrak 2a:
  JANGAN panggil `introduce_session`/`get_mapping_guess`/`confirm_mapping`/
  `run_extraction` untuk sesi gudang.
- `plugins/ingest/prompt/agent.md:348` — Kontrak 10 baru (±100 baris): tabel 7
  path baca + parameter (`wh/summary?only_stock=1`; `wh/batches?item_id&warehouse_id&near_expiry&limit&offset`),
  larangan menjumlah sendiri (`total_remaining` dari server), tabel tulis
  (hanya 2), batas keras (master tidak lewat chat, tanpa auto-create, item/gudang
  wajib ada), body JSON persis, alur 4 langkah baca→usul→tunggu→terapkan, dan
  aturan 409 = stok kurang (laporkan apa adanya, jangan ulangi payload).
- Verifikasi parameter terhadap handler asli menemukan **dua kesalahan dokumentasi
  awal** yang sudah diperbaiki: summary memakai `only_stock` (bukan
  `near_expiry_days`) dan batches memakai `near_expiry` (bukan `only_stock`).

### 8. Bug prompt yang ditemukan lewat E2E (bukan lewat unit test)

E2E chat memakai `"moved_at":"2026-11-23"` → `apply_record_actions` gagal
`warehouse: data tidak valid: moved_at harus RFC3339`. Ini murni bug kontrak
prompt (service benar menolak tanggal-tanpa-jam), tidak akan tertangkap unit
test Go. Fix: `plugins/ingest/prompt/agent.md:427` — `moved_at` opsional dan
**kosongkan** bila jam tidak disebut; bila diisi wajib RFC3339
(`2026-11-23T09:30:00+08:00`); sekaligus menegaskan `entered_at`/`expires_at`
batch justru `YYYY-MM-DD` (RFC3339 ditolak pada batch). Setelah restart runtime,
usul ulang tanpa `moved_at` → `applied`.

Gotcha operasional: system prompt dimuat **sekali** saat startup
(`INGEST_PROMPT_PATH`), jadi setiap perubahan `agent.md` wajib restart container
`bbb-data-board-ingest-runtime-1` — kalau tidak, perilaku agent "tidak berubah"
dan terlihat seperti bug kode.

### 9. Tes

- `plugins/ingest/services/segment_read_test.go:16` —
  `TestSegmentReadPathWhitelistPerSegmen` + 7 kasus gudang (whitelist lolos,
  `wh/import-batches/x` & `admins` ditolak).
- `plugins/ingest/services/record_write_test.go:18` — kasus gudang yang boleh.
- `plugins/ingest/services/record_write_test.go:95` —
  `TestSegmentWritePathWarehouseHanyaTransaksiStok`: `wh/summary`,
  `wh/batches/x`, `wh-template/import`, `wh-template/validate` → ditolak; method
  non-POST ditolak (mengunci keputusan "hanya transaksi stok").
- `plugins/ingest/services/segments_test.go:43` —
  `TestSegmentsKatalogWarehouseChatOnly`: `ChatOnly=true`, `Supported=false`,
  `Label="Gudang"`, `IsChatOnlySegment` true/false, `NormalizeSegment(" Warehouse ")`
  lolos, `TargetFields("warehouse")` tetap ditolak.

### 10. Bukti terukur

```
go build ./... && go vet ./...        → VET_OK
go test ./...                          → 26 paket ok
go test ./plugins/ingest/services/ -v  → 105 PASS

GET /api/admin/ingest/segments
  → warehouse {supported:false, chat_only:true, label:"Gudang"} (total 5 segmen)

Fail-closed (HTTP nyata via tool):
  segment_read  path=admins                → 400
  segment_read  path=wh/import-batches/x   → 400
  segment_read  method=POST                → 400
  propose       path=wh/categories         → 400
  propose       PUT  wh/batches/:id        → 400
  propose       wh-template/import         → 400
  segment_read  tanpa warehouse.read       → 403 "butuh warehouse.read …"
  propose       tanpa warehouse.write      → 403 "butuh warehouse.write …"

Bruno ingest-crud:
  baseline HEAD                    43 request / 87 test
  setelah WO-09                    64 request / 134 test
  hasil                           64 passed, 134/134 test PASS (~10,3 s)
```

### 11. E2E chat (UI `:5173`, bukti wajib 2)

Sesi `E2E gudang WO-09` (`warehouse`), master dibuat via UI/API: kategori
"Konsumsi", produk "Minyak Angin 60ml" (botol), gudang "Gudang Utama"; batch masuk
48 botol (`entered_at` 2026-11-20, `expires_at` 2026-12-20).

| Langkah | Hasil |
|---|---|
| "berapa stok minyak angin di gudang utama?" | dijawab dari `wh/summary`/`wh/batches`: **total sisa 48 liter/botol, 1 batch**, id `01a20894-4e5f-75ac-b86f-00c073666e59` (angka dikutip dari `total_remaining`, bukan karangan) |
| usul keluar 8 | kartu **"1 aksi · proposed"**; `wh/summary` tetap 48 (belum ada perubahan) |
| approve (percobaan 1) | gagal 400 `moved_at harus RFC3339` → memperlihatkan usul yang tidak disetujui memang tidak menulis data |
| approve (percobaan 2, setelah fix prompt) | kartu **"1 aksi · applied"**, "1 berhasil"; batch `qty_remaining` **48 → 40**; riwayat movement `OUT 8 Posyandu` |

Kesimpulan: agent **tidak** mengeksekusi tanpa persetujuan (59 & 58 di Bruno
memperlihatkan hal sama di level HTTP), dan angka yang dilaporkan berasal dari
tool, bukan dari prompt.

### 12. Pembersihan (AC-3)

```
wh_movements 6 · wh_import_rows 6 · wh_batches 6 · wh_import_batches 6
wh_items 6 · wh_categories 6 · wh_warehouses 6  → TRUNCATE … RESTART IDENTITY CASCADE
sisa: semua 0 baris
GET /api/admin/wh/summary → {total_items:0, total_batches:0, total_remaining:0}
GET /api/admin/wh/batches → {total:0}
```

Sesi uji segmen gudang (6 sesi: 5 `bruno-wh-chat-*` + 1 `E2E gudang WO-09`)
di-soft-delete (semua `200`; `deleted_at` terisi, 0 aktif). Akun uji
`whnoingest.wo09@test.local` & `viewer-wo09@test.local` dihapus
(`./bin/console auth delete-admin --id …`); grup `WO09 ingest tanpa warehouse`
dihapus (`DELETE /api/groups/:id` → 200).

### 13. Catatan di luar lingkup (tidak dikerjakan)

- `IngestMappingPage.tsx` masih memakai `seg.supported` untuk picker-nya; tidak
  berbahaya (halaman mapping tidak relevan untuk segmen chat-only) tetapi tidak
  ikut diseragamkan agar perubahan tetap minimal.
- Enum `target_plugin` di `introduce_session` belum memuat `warehouse` (sengaja).
- `Total: 5 segmen` di katalog berarti UI menampilkan "Gudang" sebagai pilihan
  yang sah walau `supported=false`; label bantu "(belum tersedia)" tidak muncul
  untuk gudang — sudah benar secara UX, tapi pembaca baru perlu tahu bahwa
  `chat_only` adalah sumber kebenarannya.


