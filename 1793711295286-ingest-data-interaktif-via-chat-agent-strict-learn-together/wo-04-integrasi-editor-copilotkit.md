---
title: WO-04 — Integrasi editor ↔ CopilotKit + housekeeping
description: "CP-4 final (REVISI CP-2/CP-3): agent menautkan editor via blok terstruktur + renderer, sinkronisasi dua arah via pesan sistem Go, konfirmasi generative UI dari chat; state machine backend tetap penegak; housekeeping + retensi"
status: pending
created: "2026-11-03"
---
# WO-04 — Integrasi Editor ↔ CopilotKit + Housekeeping (Checkpoint 4, FINAL)

Checkpoint: **CP-4 (terakhir)**. Prasyarat: CP-3 (WO-03) **complete (sebagian)**
— editor berdiri sendiri + rules O-20 hidup; **sisa prasyarat: §0.1/AC-0 CP-3
(berkas asli BBH) HARUS terverifikasi SEBELUM WO ini dieksekusi** agar keputusan
desain integrasi memakai hasil percakapan data nyata. Sumber: `spec.md` §6.4 +
§9 (CP-4); `spec-techstack.md` §2.4; D-10 (UI), D-11 (arsitektur), O-18/O-19/O-20.

> **⚠ REVISI (2026-11-04, pasca CP-2 & CP-3).** Pembaharuan penting dibanding
> draf awal: (1) D-10 — chat = halaman khusus `/ingest` + `/ingest/:id/chat`,
> bukan panel embed; (2) D-11 — runtime Node terpisah `sub_app/ingest_runtime/`
> (O-19), Node tipis; (3) WO-03 sudah membuat jembatan SATU ARAH chat→editor
> (tautan "Buka editor mapping" + tombol "Mapping"), jadi §A CP-4 menyempurnakan
> menjadi blok terstruktur + renderer, bukan memulai dari nol; (4) mapping v2
> + aksi `revise` + katalog `exceptions.yaml` sudah ada (WO-03 §A/§D); (5)
> compact O-18 sudah jalan di runtime — risiko konteks jauh lebih kecil;
> (6) O-15 (cakupan proxy) MOOT — proxy Go dihapus di WO-01c.

## Tujuan

Menyatukan dua permukaan yang sudah ada — panel chat (CopilotKit, WO-01/02) dan
editor mapping (WO-03) — menjadi satu pengalaman: agent menuntun, editor menyunting,
dan **state machine backend tetap satu-satunya penegak**. Plus housekeeping (daftar
sesi, retensi) agar fitur siap dipakai rutin.

## Lingkup

### A. Agent → editor: blok terstruktur + renderer (penyempurnaan WO-03 §C)

- Baseline WO-03 §C sudah ada: tautan statis "Buka editor mapping →" di halaman
  chat + tombol "Mapping" di daftar sesi. CP-4 menyempurnakan:
- Saat sesi memasuki/mengubah state relevan (`mapping_draft` setelah `revise`,
  profil berubah signifikan), **backend Go** menyuntikkan blok terstruktur
  `editor_link` (URL `/ingest/:id/map` + alasan) ke konteks/protokol jawaban —
  dihasilkan Go, BUKAN teks bebas LLM.
- CopilotKit merender blok itu sebagai tombol/kartu — pakai pola renderer tool
  (catatan WO-01b: warning "no renderer is registered for tool" — daftarkan
  renderer khusus di sini, hindari code block mentah).
- Agent TIDAK punya wewenang membuat tautan lain (deterministik, dari state).

### B. Sinkronisasi dua arah profil (chat ↔ editor) — via Go, tanpa duel state

- Perubahan mapping/rules di editor (PUT + `revise`) menyuntikkan **pesan sistem**
  ke `ingest_messages` VIA GO (pola pesan `summary` O-18): "profil diubah via
  editor: kolom X → field Y [is_encrypt] / rule `visit_missing_phone` skip" —
  konteks percakapan selalu profil terbaru.
- Agent tidak boleh menebak ulang kolom yang sudah dikonfirmasi; tebakan hanya
  untuk kolom baru/belum terkonfirmasi (system prompt `agent.md` diperbarui).
- Arah editor→chat adalah yang baru; arah chat→editor sudah berlaku sejak WO-03
  (profil sama, tanpa salinan).
- **Keputusan terkunci & rules tidak ikut "diingat" oleh model**: profil Go =
  source of truth; O-18 sudah menyuntikkan ulang profil tiap request — pesan
  sistem ini hanya narasi, bukan sumber kebenaran.

### C. Konfirmasi & ekstraksi dari chat (generative UI)

- Kartu ringkasan di chat (n kolom mapped, gate yang belum lolos dari API gate
  4 status WO-03, kebijakan dedup terpilih) dengan tombol **"Konfirmasi mapping"**
  dan **"Ekstrak"** → memanggil action API yang sama dengan editor
  (`POST .../mapping/confirm`, `POST .../mapping/extract`) DENGAN guard RBAC
  + `dedup_policy` wajib.
- Klien TIDAK memanggil confirm/ekstraksi lewat tool-calling LLM; backend
  menegakkan (percobaan tool-calling menulis profil/confirm → 403 + audit).
- Tool agent (`get_mapping_guess`/`confirm_mapping`/`run_extraction`, pola
  WO-01b) tetap ada untuk alur chat murni; kartu generative UI adalah jalur
  deterministik tambahan, bukan pengganti gate.

### D. Housekeeping (penyempurnaan; halaman daftar sudah ada sejak WO-01)

- Daftar `/ingest` sudah ada (WO-01): tambahkan kolom state machine + plugin
  target + tombol kontekstual per state ("Lanjut chat" / "Buka editor" /
  "Konfirmasi" / "Ekstrak") — baca dari API gate, bukan tebak di klien.
- **Delete sesi (keputusan owner, 2026-11-06): soft delete + opsi hard delete.**
  - **Soft delete** = default: kolom `deleted_at` di `ingest_sessions`
    (migrasi baru), sesi terhapus hilang dari daftar default; sesi terhapus
    masih bisa dilihat/ dipulihkan lewat filter eksplisit.
  - **Hard delete** = aksi kedua, dijalankan SETELAH konfirmasi eksplisit:
    hapus fisik baris sesi + `ingest_messages` + `ingest_uploads` (dan berkas
    di storage) milik sesi itu.
  - **Guard `ingest.write` + audit** untuk kedua aksi (pola D-6); klien tidak
    menghapus lewat tool-calling LLM — hanya action API.
  - Sesi ber-state `extracted` menyimpan profil (inti learn-together, D-8; O-14
    mengusulkan profil disimpan tak terbatas) → **hard delete sesi extracted
    WAJIB konfirmasi eksplisit berlabel risiko kehilangan profil**; soft delete
    tetap diizinkan.
  - Endpoint usulan: `DELETE /api/admin/ingest/sessions/:id` (soft, default)
    + `?mode=hard` / aksi terpisah untuk hard delete; restore = aksi API
    eksplisit (`POST .../sessions/:id/restore`).
- **Checkbox multi-select di daftar `/ingest` (keputusan owner, 2026-11-06):**
  pilih banyak sesi → aksi **bulk delete** (soft delete) lewat satu endpoint
  batch (usulan: `POST /api/admin/ingest/sessions/bulk-delete`), dengan modal
  konfirmasi berisi jumlah sesi terpilih; hasil per-sesi dilaporkan (sukses /
  gagal / ditolak karena state terkunci). Checkbox hanya di daftar, bukan di
  halaman chat/editor.
- Retensi & cleanup: sesi/file kedaluwarsa dihapus sesuai keputusan **O-14**
  (masih menunggu keputusan owner — diisi SEBELUM eksekusi; default:
  konfigurabel + TODO + audit job periodik). Hapus manual (di atas) dan job
  retensi O-14 HARUS memakai jalur penghapusan yang sama supaya tidak saling
  melanggar aturan soft/hard delete.
- ~~O-15 (cakupan proxy)~~ **MOOT** — proxy Go dihapus di WO-01c (keputusan
  MCP/D-11); tidak ada tindak lanjut.
- Housekeeping kecil: hapus tambalan `provider-compat.ts` JIKA gateway sudah
  benar (cek dulu; masih dipertahankan bila belum); rapikan komentar compose;
  README how-to pemakaian rutin.

## Acceptance Criteria

- AC-0 Prasyarat: §0.1/AC-0 CP-3 (berkas asli BBH via chat) TERVERIFIKASI dan
  terekam di dok WO-03 sebelum WO-04 dieksekusi.
- AC-1 Pada state yang tepat, blok `editor_link` (dari Go) dirender CopilotKit
  sebagai tombol/kartu (bukan code block); klik → editor membuka profil sesi
  yang sama.
- AC-2 Perubahan mapping/rules via editor muncul sebagai pesan sistem di chat
  (via Go) dan jawaban agent berikutnya konsisten dengan profil baru; agent
  tidak menebak ulang kolom terkonfirmasi.
- AC-3 Konfirmasi/ekstraksi dari kartu generative UI memakai profil terbaru;
  semua gate (4 status) + `dedup_policy` wajib + penegakan rules identik dengan
  jalur editor.
- AC-4 Klien/tool-calling tidak bisa menulis profil atau memicu confirm/ekstraksi
  langsung (403 + audit).
- AC-5 Tidak ada jalur UI (chat maupun editor) yang bisa melewati gate — state
  machine backend tetap penegak tunggal.
- AC-6 Daftar `/ingest` menampilkan state + aksi kontekstual per sesi (baca API,
  bukan tebakan klien).
- AC-7 Retensi/cleanup sesi & file berjalan sesuai keputusan O-14 (jika owner
  sudah memutuskan); tanpa keputusan, implementasi default konfigurable +
  ditandai TODO + audit.
- AC-8 `npm run build` hijau; `go vet`/`go test` hijau; swagger + Bruno untuk
  protokol baru; how-to pemakaian rutin di README plugin ingest.
- AC-9 (jika gateway belum benar) `provider-compat.ts` tetap berfungsi pasca
  integrasi — chat/tool tidak regresi (regresi nol pola AC-0 WO-02).
- AC-10 Delete sesi & checkbox: soft delete default + hard delete hanya lewat
  konfirmasi eksplisit (hard delete sesi `extracted` berlabel risiko profil);
  checkbox multi-select → bulk delete soft via endpoint batch; kedua aksi
  ber-guard `ingest.write`, ter-audit, sesi terhapus hilang dari daftar default,
  dan regresi nol pada daftar/aksi kontekstual (AC-6).

## Non-tujuan

- Fitur baru di luar rantai ingest (ekspor laporan, analitik sesi, multi-file dalam
  satu sesi).
- Mengubah kebijakan deidentify atau aturan fail-closed yang sudah terkunci (D-1..D-9).

## Non-tujuan

- Fitur baru di luar rantai ingest (ekspor laporan, analitik sesi, multi-file dalam
  satu sesi).
- Mengubah kebijakan deidentify atau aturan fail-closed yang sudah terkunci (D-1..D-11).
- Rule baru lintas plugin lain (hsh/sec) — skema generik saja (bawaan WO-03).

## Risiko & catatan

- Sinkronisasi dua arah jangan menciptakan duel state: **pesan sistem hanya masuk
  ke chat (via Go), agent tidak pernah menulis profil** — profil hanya ditulis
  oleh action API (D-11).
- Ukuran konteks percakapan SUDAH ditangani compact O-18 (runtime Node,
  `INGEST_LLM_CONTEXT_LIMIT` 70%, `[LOCKED]` tidak diringkas) — jangan duplikasi
  mekanisme ringkas baru; cukup pastikan pesan sistem editor ikut pola penyimpanan
  via Go.
- Renderer generative UI = area baru CopilotKit — uji dulu pola renderer tool
  (catatan WO-01b) sebelum membangun kartu konfirmasi; fallback: code block +
  tombol di luar pesan.
- `provider-compat.ts` tetap ketergantungan eksternal (gateway) — AC-9 menjaga
  regresi nol; hapus begitu gateway benar.
- Sisa open question owner (O-3..O-9/O-13/O-16) tidak menghalangi eksekusi —
  default sudah terpakai; review sekalian di akhir CP-4 (review akhir fitur).

---
Setelah CP-4 lulus AC, fitur ingest interaktif dianggap **complete**; review akhir
(owner) menilai hasil pilot `visit` sebelum perluasan ke plugin lain.
