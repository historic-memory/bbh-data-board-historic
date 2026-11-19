---
title: "WO-02 — Listing UI: tabel rejected + checkbox scope + edit manual + tombol Analisa"
description: "Panel Baris Rejected di IngestMappingPage: checkbox default P-8, edit manual inline, tombol Analisa (N) memicu loop agent, Setujui per baris/kategori/semua, link perbaiki via chat; persisten O-25"
status: pending
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - frontend
    - listing
---
# WO-02 — Frontend: listing UI + checkbox scope + edit manual + tombol Analisa

Sumber: SPEC `spec.md` topic `1795083433686`. Prasyarat: WO-01
(snapshot + tool T-1/T-2/T-4 hidup). Status: **pending (menunggu
WO-01 selesai)**.

## Lingkup

### A. Panel "Baris Rejected" di `IngestMappingPage`

1. Tabel: `☑` checkbox (default dari flag P-8 backend — format ☑ /
   kelengkapan ☐) · RowNo · kolom bermasalah · nilai terdeidentify ·
   `violates` · usulan AI · status (menunggu/diusulkan/diedit user/
   disetujui/perlu keputusan manusia).
2. **Edit manual inline** per baris → masuk proposal yang sama
   (`edited_by: user`) — jalur manual & AI bermuara ke satu proposal store.
3. Tombol **"Analisa (N)"** → simpan scope row_no terpilih ke `analyze`
   → tutup/keluar mapping editor → auto-submit SATU pesan basa-basi ke
   chat AI (mis. "Tolong analisa baris rejected yang barusan
   disimpan") — TANPA data di pesan: session id & snapshot sudah ada
   di sesi, AI mengambil via tool (P-9); loop agent berjalan atas
   baris terpilih (P-7). Pesan single-shot: refresh tidak mengirim ulang.
4. Tombol **"Setujui"** per baris / per kategori / semua →
   `apply_fixes` (T-3, WO-03) dengan `confirmed_by_user`.
5. Tombol **"perbaiki via chat"** per baris/kelompok → buka chat dengan
   konteks baris (blok terstruktur backend, pola `editor_link` WO-04).
6. Regenerasi listing setiap apply/edit (re-scan otomatis, WO-03) —
   snapshot persisten (O-25) → reload tetap ada.
7. PermissionGate `ingest.write` untuk tombol mutasi (pola editor
   existing); RBAC tetap.

## Acceptance Criteria

- AC-1 Listing tampil dari snapshot WO-01 dan persisten (reload tetap).
- AC-2 Checkbox mengontrol scope Analisa; default sesuai flag P-8.
- AC-3 Edit manual inline masuk proposal (`edited_by: user`) dan tampil
  di kolom usulan/status.
- AC-4 Tombol "Analisa (N)" menyimpan scope (analyze), menutup editor,
  lalu chat otomatis terkirim SATU pesan basa-basi (single-shot, tidak
  terkirim ulang saat refresh) dan loop agent berjalan atas baris
  terpilih; hasil proposal muncul di kolom usulan AI.
- AC-5 Tombol Setujui memanggil `apply_fixes` dengan
  `confirmed_by_user` (penerapan diuji penuh di WO-03).
- AC-6 Link "perbaiki via chat" membuka chat dengan konteks baris.
- AC-7 RBAC: viewer tanpa `ingest.write` hanya melihat (tombol mutasi
  tersembunyi); tanpa token 401.
- AC-8 `tsc --noEmit` + `npm run build` hijau; Playwright smoke via
  Tailscale (`http://100.125.175.34:5173`); **Log Implementasi (Dev)**.

## Non-tujuan

- Penerapan proposal (T-3, CSV perbaikan, re-scan) — WO-03.
- Blok chat `rejected_listing` — WO-03 (link tombol boleh dibuat
  statis dulu, disempurnakan saat WO-03).

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)
