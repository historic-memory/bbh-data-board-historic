---
title: WO-04 — Integrasi editor ↔ CopilotKit + housekeeping
description: "CP-4 final: agent menautkan link editor, sinkronisasi dua arah profil, konfirmasi dari panel chat; state machine backend tetap penegak"
status: pending
created: "2026-11-03"
---
# WO-04 — Integrasi Editor ↔ CopilotKit + Housekeeping (Checkpoint 4, FINAL)

Checkpoint: **CP-4 (terakhir)**. Prasyarat: CP-3 (WO-03) selesai — editor berdiri
sendiri dan bisa memicu ekstraksi. Sumber: `spec.md` §6.4 (chat UI) + §9 (CP-4);
`spec-techstack.md` §2.4 (protokol jawaban agent).

## Tujuan

Menyatukan dua permukaan yang sudah ada — panel chat (CopilotKit, WO-01/02) dan
editor mapping (WO-03) — menjadi satu pengalaman: agent menuntun, editor menyunting,
dan **state machine backend tetap satu-satunya penegak**. Plus housekeeping (daftar
sesi, retensi) agar fitur siap dipakai rutin.

## Lingkup

### A. Agent → link editor (backend yang memicu, bukan LLM improvisasi)

- Saat sesi memasuki `mapping_draft` (atau mapping berubah signifikan), protokol
  jawaban agent diperluas dengan blok terstruktur `editor_link` (URL `/ingest/:id/map`) —
  dihasilkan backend saat menyusun konteks LLM, bukan teks bebas dari model.
- CopilotKit merender blok itu sebagai tombol/kartu "Buka editor mapping".
- Agent TIDAK punya wewenang membuat tautan lain (deterministik, dari state).

### B. Sinkronisasi dua arah profil (chat ↔ editor)

- Perubahan mapping di editor menyuntikkan **pesan sistem** ke `ingest_messages`
  ("profil diubah via editor: kolom X → field Y [is_encrypt]"), sehingga konteks
  percakapan selalu memakai profil terbaru.
- Agent **tidak boleh menebak ulang** kolom yang sudah dikonfirmasi; tebakan hanya
  untuk kolom baru/belum terkonfirmasi.
- Arah sebaliknya: perubahan via chat (kalimat user "kolom A skip") tetap terlihat
  di editor saat dibuka (profil sama — sudah berlaku sejak WO-03; di checkpoint ini
  yang baru adalah arah editor → chat).

### C. Konfirmasi dari panel chat (generative UI)

- Panel chat menampilkan kartu ringkasan (n kolom mapped, m gate belum lolos) dengan
  tombol **"Konfirmasi mapping"** dan **"Ekstrak"** → memicu action API yang sama
  dengan editor (`POST .../mapping/confirm`, endpoint ekstraksi).
- Klien TIDAK memanggil confirm/ekstraksi lewat tool-calling LLM; backend menegakkan
  (percobaan tool-calling menulis profil/confirm → 403 + audit).

### D. Housekeeping

- Halaman daftar `/ingest`: tabel sesi (status state machine, plugin target, waktu,
  pembuat) + tombol "Lanjut chat" / "Buka editor" sesuai state.
- Retensi & cleanup: sesi dan file upload kedaluwarsa dihapus sesuai keputusan
  **O-14** (diisi owner sebelum WO ini dieksekusi); job cleanup periodik + audit.
- O-15 (cakupan proxy) ditindaklanjuti sesuai keputusan owner.

## Acceptance Criteria

- AC-1 Pada state yang tepat, jawaban agent menyertakan blok `editor_link` yang
  dirender CopilotKit sebagai tombol; klik → editor membuka profil sesi yang sama.
- AC-2 Perubahan mapping via editor muncul sebagai pesan sistem di chat dan jawaban
  agent berikutnya konsisten dengan profil baru.
- AC-3 Konfirmasi ekstraksi dari panel chat memakai profil terbaru; `DecisionClassifier`
  dan semua gate tetap jalan identik dengan jalur editor.
- AC-4 Klien/tool-calling tidak bisa menulis profil atau memicu confirm/ekstraksi
  langsung (403 + audit).
- AC-5 Tidak ada jalur UI (chat maupun editor) yang bisa melewati gate — state machine
  backend tetap penegak tunggal.
- AC-6 Halaman `/ingest` menampilkan daftar sesi lengkap dengan status + aksi lanjut.
- AC-7 Retensi/cleanup sesi & file berjalan sesuai keputusan O-14 (jika owner sudah
  memutuskan); tanpa keputusan, implementasi default konfigurable + ditandai TODO.
- AC-8 `npm run build` hijau; `go vet`/`go test` hijau; swagger + Bruno untuk
  endpoint/protokol baru; catatan pemakaian rutin (how-to) di README plugin ingest.

## Non-tujuan

- Fitur baru di luar rantai ingest (ekspor laporan, analitik sesi, multi-file dalam
  satu sesi).
- Mengubah kebijakan deidentify atau aturan fail-closed yang sudah terkunci (D-1..D-9).

## Risiko & catatan

- Sinkronisasi dua arah jangan menciptakan duel state: **pesan sistem hanya masuk ke
  chat, agent tidak pernah menulis profil** — profil hanya ditulis oleh action API.
- Jaga ukuran konteks percakapan: ringkas riwayat saat sesi panjang (mis. ganti
  lampiran lama dengan ringkasan keputusan), tanpa menghilangkan keputusan terkunci.

---
Setelah CP-4 lulus AC, fitur ingest interaktif dianggap **complete**; review akhir
(owner) menilai hasil pilot `visit` sebelum perluasan ke plugin lain.
