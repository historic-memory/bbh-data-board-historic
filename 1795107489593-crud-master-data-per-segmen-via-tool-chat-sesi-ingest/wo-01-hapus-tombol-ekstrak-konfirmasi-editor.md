---
title: "WO-01 — Hapus tombol Ekstrak & Konfirmasi dari editor mapping (aksi jalan via chat AI)"
description: "Hapus tombol 'Ekstrak sekarang' (kartu Ekstrak: dedup + dry run ikut hilang) dan 'Konfirmasi & kunci mapping' di IngestMappingPage — konfirmasi/ekstraksi cukup lewat tool chat AI (confirm_mapping/run_extraction); Simpan draft tetap; prasyarat data dedup/dry-run pindah ke alur chat"
status: complete
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - frontend
    - ux
---
# WO-01 — Hapus tombol Ekstrak & Konfirmasi dari editor mapping

Sumber: owner 2026-11-20 — "sesuai rencana kita pakai tools AI chat".
Topic `1795107489593` (extends `1795083433686`). Status: **progress**
(handoff ke dev 2026-11-20 setelah SPEC disetujui owner).
Patch frontend kecil (pola patch langsung, bukan fitur besar).

## Handoff Manager → Dev (2026-11-20)

- SPEC topic ini **disetujui owner** — mulai eksekusi dok ini.
- Urutan kerja: Lingkup A (hapus tombol/kartu + state) → A.4 modal
  Simpan draft (WAJIB, komponen `ui/Modal` existing) → B verifikasi
  alur dedup/dry-run di chat (bila jalur tanya dedup belum ada di
  prompt, tambahkan kecil — JANGAN restore tombol) → C verifikasi
  build + smoke Tailscale.
- Aturan pinned: isi **Log Implementasi (Dev)** bertahap per bagian;
  lapor balik ke Manager dengan ringkasan + hash commit saat selesai
  untuk direview (pola topic `1795083433686`).

## Latar

Editor mapping (`IngestMappingPage.tsx`) masih memuat dua tombol aksi
jalur manual: **"Konfirmasi & kunci mapping"** (`confirm()` →
`confirmMapping`) dan **"Ekstrak sekarang"** (`extract()` →
`extractMapping` dengan pilih dedup + dry run). Sejak tool chat AI
(`confirm_mapping`/`run_extraction`) matang, dua jalur ini redundan dan
mempertahankan dua sumber kebenaran. Owner memutuskan: **hapus dua-duanya**
— aksi konfirmasi/ekstraksi cukup lewat chat AI. Tambahan owner (2026-11-20):
**tombol "Simpan draft" diberi modal prompt untuk kembali ke chat** —
jalur editor adalah persiapan, jalur eksekusi adalah chat.

## Lingkup

### A. Frontend (`sub_app/webapp/src/pages/admin/ingest/`)

1. Hapus tombol **"Konfirmasi & kunci mapping"** di footer
   `IngestMappingPage.tsx` (blok `Button ... confirm()`).
2. Hapus kartu **"Ekstrak"** di sidebar (judul, pilihan kebijakan dedup,
   checkbox dry run, tombol "Ekstrak sekarang") beserta state/handler yang
   hanya dipakai keduanya (`busy 'confirm'|'extract'`, `dedup`, `dryRun`,
   `confirm()`, `extract()`, `gateReady` yang hanya dipakai tombol).
3. **Simpan draft TETAP ADA** — jalur simpan mapping manual tetap dipakai
   (juga dipakai konfirmasi per baris tebakan WO-11).
4. **Modal prompt setelah Simpan draft (owner 2026-11-20)** — sukses simpan
   menampilkan modal (komponen `ui/Modal`, pola Modal hapus existing) dengan
   pilihan: **"Kembali ke chat"** → navigasi `/ingest/:id/chat`, atau
   **"Lanjut edit"** → tetap di editor (modal tutup). Notice success pindah
   isi ke modal (jangan dobel). Simpan draft TIDAK memicu navigasi otomatis
   selain lewat pilihan user (konsisten D-7: keputusan selalu manusia).
5. Perbaiki teks panduan di halaman yang menyebut tombol terkait (mis.
   "Tombol Ekstrak baru aktif setelah semua gate lolos" pada notice
   `confirm()` yang dihapus, dan subtitle kartu Ekstrak) → arahkan ke
   sesi chat AI ("konfirmasi & ekstraksi dilakukan lewat sesi chat").
6. Gate panel (daftar gate) TETAP TAMPIL — masih kebutuhan informasi.
   Panel "Pengecualian (aturan)" tetap (input rule manual tetap valid).

### B. Alur dedup / dry-run (perhatian khusus)

- Pilihan **kebijakan dedup** dan **dry run** sebelumnya ada di kartu
  Ekstrak yang dihapus. Verifikasi alur chat: tool `run_extraction`
  (Node `copilot-runtime.ts` + backend `RunExtraction`) menerima
  `dedup_policy`/`dry_run` dari argumen tool — agent menanyakan user lalu
  mengirim nilai. **Prasyarat**: agent prompt (`agent.md`) sudah
  menginstruksikan konfirmasi kebijakan dedup (checklist topic induk
  AC-4 gate Aturan: 177 baris tanpa HP → agent WAJIB bertanya). Jika
  ternyata belum ada jalur itu, TIDAK BOLEH diabaikan: catat temuan dan
  tambahkan di prompt (kecil), bukan restore tombol.

### C. Verifikasi

- `npx tsc -b` + `npm run build` hijau; tidak ada referensi mati
  (`confirmMapping`/`extractMapping` import dihaluskan — bila tidak dipakai
  lagi di file lain, hapus import).
- Smoke browser Tailscale `http://100.125.175.34:5173`: editor mapping tanpa
  dua tombol; daftar gate + panel rejected + Simpan draft tetap ada; alur
  konfirmasi → ekstraksi lewat chat AI tetap jalan end-to-end (sesi uji
  baru); dry-run via chat terbukti.
- Regresi: alur upload → tebakan → Setujui (WO-11) → analisa (WO-02) tidak
  berubah.

## Acceptance Criteria

- AC-1 Tombol "Konfirmasi & kunci mapping" dan kartu "Ekstrak" (dengan
  dedup + dry run + tombolnya) tidak ada lagi di editor mapping.
- AC-2 Simpan draft tetap ada dan berfungsi (regresi WO-11: Setujui
  tebakan → PUT mapping jalan); sukses simpan menampilkan modal pilihan
  "Kembali ke chat" (navigasi) / "Lanjut edit" (tetap); tidak ada
  navigasi otomatis tanpa pilihan user.
- AC-3 Konfirmasi & ekstraksi end-to-end hanya via chat AI (tool
  `confirm_mapping` + `run_extraction`) — terbukti satu sesi uji penuh.
- AC-4 Kebijakan dedup & dry run tetap bisa dipilih user DI CHAT (agent
  menanyakan; bukan default diam-diam) — bila belum ada, tambah ke prompt.
- AC-5 Gate panel + panel rejected + panel aturan tetap tampil.
- AC-6 `tsc -b` + `npm run build` hijau; smoke browser Tailscale;
  **Log Implementasi (Dev)** di dok ini.
- AC-7 Modal baru memakai komponen `ui/Modal` existing (tanpa dependensi
  baru); keyboard Escape/click-luar menutup modal = "Lanjut edit".

## Non-tujuan

- CRUD master data per segmen (tool M-1..M-4) — WO-02/topic ini, desain
  draf di `notes.md`.
- Perubahan backend/endpoint (`confirm_mapping`/`run_extraction` tetap).
- Menghapus halaman editor atau panel lain.

## Catatan Implementasi (Dev)

Format: `berkas:baris` — apa yang dilakukan — kenapa (ref AC/D-x/gotcha).

1. `sub_app/webapp/.../ingest/IngestMappingPage.tsx:99` — siklus tipe `busy`
   dipangkas jadi `'load' | 'save' | 'resample' | ''` (buang `'confirm'`/
   `'extract'`) — AC-1: dua state itu hanya melayani tombol yang dihapus.
2. `IngestMappingPage.tsx:~100` — state `dedup`/`dryRun`/`gateReady` dihapus —
   AC-1; `gateReady` sempat hanya dipakai untuk mengaktifkan tombol Ekstrak,
   sedangkan `gateReady`-nya sendiri tidak lagi dibutuhkan di footer.
3. `IngestMappingPage.tsx` import — buang `DEDUP_POLICIES`, `confirmMapping`,
   `dedupPoliciesFor`, `extractMapping` dari `src/api/ingest.ts` — AC-6 "tidak
   ada referensi mati"; API-nya sendiri TIDAK dihapus (dipakai tempat lain).
4. `IngestMappingPage.tsx:87` — tambah `const navigate = useNavigate()` —
   AC-2 butuh navigasi ke `/ingest/:id/chat` dari modal.
5. `IngestMappingPage.tsx:102` — state baru `savePromptOpen` — AC-2/AC-7:
   satu sumber kebenaran untuk visibilitas modal Simpan draft.
6. `IngestMappingPage.tsx:232-256` — `persistColumns(next, successNotice,
   promptChat=false)`; `save = () => persistColumns(payloadColumns, '', true)`
   — AC-2: satu jalur PUT mapping untuk ketiga aksi (setuju tebakan / setuju
   semua / Simpan draft), flag `promptChat` yang membedakan modal vs notice
   inline. Aksi tebakan tetap notice inline supaya user tidak dipindahkan dari
   editor saat masih memetakan kolom lain.
7. `IngestMappingPage.tsx:252` — sukses simpan draft → `setSavePromptOpen(true)`
   dan TIDAK set notice → AC-2 "jangan dobel" (notice pindah isi ke modal).
8. `IngestMappingPage.tsx:608-621` — kartu sidebar "Ekstrak" diganti kartu
   "Langkah berikutnya" berisi teks panduan konfirmasi/ekstraksi lewat chat +
   `Link` "Buka chat sesi" (`data-testid="ingest-go-chat"`) — AC-1 + §A.5:
   teks lama yang menyebut tombol dihapus diganti arahan ke sesi chat.
9. `IngestMappingPage.tsx:731-733` — footer kini hanya "Simpan draft" — AC-1;
   "Konfirmasi & kunci mapping" hilang beserta `confirm()`.
10. `IngestMappingPage.tsx:741-765` — `<Modal open={savePromptOpen}
    title="Draft tersimpan" size="sm" onClose={...}>` dengan 2 aksi:
    "Lanjut edit" (`ingest-save-continue`) dan "Kembali ke chat"
    (`ingest-save-go-chat` → `navigate('/ingest/${id}/chat')`) — AC-2/AC-7.
    `onClose` = "Lanjut edit" sehingga Escape/klik-luar tidak menavigasi
    (konsisten D-7: keputusan selalu manusia, tanpa navigasi otomatis).
11. `IngestMappingPage.tsx` doc-comment header — langkah alur diperbarui jadi
    "4. simpan draft → lanjut di chat" + ref WO-01 topic `1795107489593` — agar
    pembaca berikutnya tidak mencari tombol konfirmasi di editor.
12. `plugins/ingest/prompt/agent.md:89` — tambah bullet "Tombol konfirmasi/
    ekstraksi TIDAK ADA di editor; konfirmasi & ekstraksi adalah tugasmu di
    chat" — AC-4: agent tidak lagi mengarahkan user ke tombol yang sudah dihapus.
13. `plugins/ingest/prompt/agent.md:86` — bullet "dry run juga keputusan
    pengguna; jangan simpulkan dari jawaban dedup" — AC-4: prasyarat dedup/
    dry-run pindah dari kartu Ekstrak ke percakapan chat.
14. `plugins/ingest/prompt/agent.md:61` — tegaskan bentuk `mapping` = objek peta
    `{"kolom":"ref field"}` — temuan smoke: agent sempat mengirim daftar
    `[{source,target}]` → 400 "profil tidak valid".
15. `plugins/ingest/prompt/agent.md:67` — bullet baru bentuk dokumen v2 +
    `kind` untuk kolom PII — gotcha: bentuk peta legacy tidak membawa `kind`,
    sehingga backend menolak `400 … melonggarkan policy bawaan … wajib
    terdeidentifikasi` untuk `visit_visitors.full_name`/`phone_raw`
    (`plugins/visit/schema/fields.yaml:35,95`). `flags.is_encrypt` TIDAK
    dibaca validator — jangan sarankan itu ke agent.
16. `sub_app/ingest_runtime/tool-schema.ts:40` — helper baru
    `loosenFreeFormFields(schema, paths)` — AKAR temuan AC-3/AC-4:
    `convertJsonSchemaToZodSchema` (@copilotkit/runtime/v2) membuang
    `additionalProperties` sehingga `mapping`/`rules`/`flags`/`kinds` jadi
    `{type:object, properties:{}, additionalProperties:false}` → model tidak
    boleh mengisi satu kunci pun, dan `confirm_mapping` selalu gagal.
17. `tool-schema.ts:44` — `loosen()` mengelompokkan dot-path, membuka
    `ZodOptional._def.innerType`, dan membangun ulang lewat `schema.extend()`;
    `openPath()` menangani `ZodArray` (item dibangun ulang) dan daun bebas
    memakai `.passthrough()` — dipilih daripada `.catchall(z.any())` karena
    round-trip JSON schema tetap `additionalProperties:true`/`{type:string}`.
18. `sub_app/ingest_runtime/copilot-runtime.ts:203-236` — `confirm_mapping`
    membungkus `mapping`/`rules`/`flags`; `:181-200` `get_mapping_guess`
    membungkus `kinds`; `:289` `propose_fixes` membungkus `fixes.values`
    (pola sama) — tanpa ini tool yang sama masih terkunci.
19. `sub_app/ingest_runtime/tool-schema.test.ts` — 3 test (peta bebas diterima,
    peta bersarang `fixes.values`, field bertipe tetap menolak tipe salah) +
    `sub_app/ingest_runtime/package.json:10` menambah file itu ke `npm test` —
    AC-6; tanpa test ini regresi skema tidak terdeteksi.
20. Verifikasi smoke (Tailscale `http://100.125.175.34:5173`): sesi uji
    `wo01-smoke-1795126932` — editor tanpa 2 tombol, gate panel + "Pengecualian
    (aturan)" + "Simpan draft" tetap; modal Escape/klik-luar = "Lanjut edit",
    "Kembali ke chat" → `/chat`; alur chat `confirm_mapping` 200 → state
    `mapping_confirmed` → `run_extraction`(`dedup_policy=time_only`,
    `dry_run=true`) 200 `status: staged` (batch `01a1f5f0-…`) — AC-3/AC-4
    terbukti end-to-end. Catatan lingkungan (bukan bug WO ini): sesi baru dari
    `uploaded` butuh aksi `introduce` yang hanya dijalankan editor
    (`plugins/ingest/services/mapping.go:786`) — untuk uji chat murni, gate itu
    dimajukan lewat `POST /admin/ingest/sessions/:id/transition`.

## Review Manager (2026-11-20) — LULUS ✅

Verifikasi independen (bukan hanya percaya laporan):

- `git show 111bf03` + `8d0eafb` — isi commit sesuai lingkup; berkas di luar
  lingkup TIDAK ikut.
- Grep `IngestMappingPage.tsx` — tidak ada lagi `confirmMapping`/
  `extractMapping`/state `dedup`/`dryRun`/`gateReady`; sisa kecocokan hanya
  komentar arahan ke chat (AC-1).
- Modal pasca simpan (baris ±741-765) memakai `ui/Modal` existing, dua tombol
  bertanda `ingest-save-continue`/`ingest-save-go-chat`, `onClose` = Lanjut
  edit, tanpa navigasi otomatis (AC-2/AC-7).
- `tool-schema.ts` (`loosenFreeFormFields`) + pembungkusan 3 tool di
  `copilot-runtime.ts` + 3 test regresi terpasang di `npm test` — Manager
  jalankan sendiri di container `ingest-runtime`: **20/20 PASS** (AC-6).
- Smoke browser sendiri (sesi `wo03-smoke-1795104921/map`, reload): footer
  hanya "Simpan draft"; kartu "Langkah berikutnya" + Gate + "Pengecualian
  (aturan)" + panel Baris Rejected ("Analisa"/"Setujui semua") tetap tampil
  (AC-1/AC-5).
- Log Implementasi: 20 entri, format `berkas:baris` + kenapa dengan ref
  gotcha/AC — kualitas baik (aturan owner `7e1a197` terpenuhi).

### Putusan 3 temuan dev

1. **Bentuk peta legacy tanpa `kind`** (field `encrypt:true` ditolak
   `ErrPolicyRelaxed`) → DITERIMA SEBAGAI BACKLOG, tidak blocker. Prompt sudah
   mengarahkan bentuk v2 + `kind`; usulan backend "default `kind` dari
   `fields.yaml` saat kolom legacy" dicatat sebagai kandidat perbaikan kecil
   (kandidat: masuk WO-05 housekeeping atau patch kecil terpisah — jangan
   dikerjakan tanpa WO).
2. **Gate `introduced` hanya via editor** (`ActionIntroduce`) → DIPERTAHANKAN
   "editor = persiapan" (konsisten arah owner: chat = jalur eksekusi).
   Peningkatan "sesi pure-chat dari `uploaded`" dicatat sebagai OPEN QUESTION
   ke owner: (a) tool kecil `introduce_session` di Node, atau (b) auto-introduce
   saat pesan pertama agent. Menunggu keputusan owner; tidak blocker WO-02..05.
3. **Sesi uji dev `wo01-smoke-1795126932`** → boleh dibersihkan via soft delete
   daftar sesi (WO-04a) kapan saja; tidak menahan apa pun.

Status WO: **complete**. Berikutnya: WO-02 (`segment_read`) siap eksekusi.
