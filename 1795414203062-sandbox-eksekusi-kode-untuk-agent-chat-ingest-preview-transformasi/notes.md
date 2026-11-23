---
title: notes
description: "Bukti masalah, keputusan arsitektur, dan alternatif desain dari meeting 2026-11-24."
status: progress
created: "2026-11-23"
updated: "2026-11-24"
tags: [notes]
---

# Catatan

## Insiden & bukti
- Salah-diagnosa agent chat 3× (sesi d1f3d30b, 873db24a): "deadlock"/"deterministik gagal" tanpa cek state. Kontrak 9 (topic 1795182594048) mengurangi, tidak menyembuhkan akar: agent tak bisa membuktikan transformasi sendiri.
- Tumpang-tindih tool: `analyze_session` hanya evaluasi rules; validasi enum/tanggal/berat berjalan di lapisan import — scan bersih ≠ ekstraksi lolos (dibuktikan via API + DB + berkas, 2026-11-23).
- Bukti rule-loss: profil v6 `rules:{}` (confirmed 14:57) — backlog terpisah.

## Keputusan arsitektur (2026-11-23)
- Sandbox self-hosted (PII tidak keluar VPS); cloud sandbox ditolak.
- Runtime utama: Pyodide (WASM) di `ingest_runtime`; fallback gVisor sidecar = WO terpisah.
- Fail-closed: sandbox hanya proposer; penyimpanan tetap via `apply_fixes`/validasi Go.
- Guard: timeout 10s, cap memori/output, kuota 10 exec/sesi, no-net/fs, audit semua exec.

## Alternatif desain dari meeting 2026-11-24 (BARU)
User mengusulkan memakai agent umum "1 paket" (mis. OpenCode — punya built-in tools: edit file, shell, exec kode) sebagai agent manipulasi dokumen ingest, alasan: lebih mudah, tak perlu bangun sandbox+tools sendiri.
- Klarifikasi Manager: yang "1 paket" itu agent OpenCode-nya, bukan protokol ACP (Agent Client Protocol = hanya jembatan editor↔agent).
- **Risiko jika dipakai langsung**: akses shell/filesystem luas → bisa bypass fail-closed (edit CSV langsung, lewati validasi Go/state machine/audit); PII keluar ke LLM provider; non-deterministik; permission tidak terhubung RBAC.
- **Jalan tengah yang layak ditimbang**: pola "proposer dalam sandbox ketat" — agent umum di container terbatas (no-network, hanya berkas sesi read-only, no secrets), keluarannya *usulan/diff* yang tetap disetujui user lalu dieksekusi lewat pipeline Go. Kemudahan "1 paket" dipertahankan, fail-closed tetap.
- Status: ALTERNATIF kandidat — menyaingi/melengkapi Pyodide (WO-01). Keputusan final menunggu perbandingan biaya/risiko: Pyodide = kontrol penuh & kecil, agent umum = fitur kaya tapi butuh hardening container.

## Info baru dari owner (2026-11-24): donis-agent punya fungsi deterministik
Owner mengklarifikasi: **donis-agent adalah proyek milik owner sendiri**, dan owner sedang garap **fungsi teks deterministik di sisi provider**. Dampaknya ke tabel risiko di atas:
- Risiko "non-deterministik" agent umum **menurun** jika donis-agent menjamin output deterministik — argumen kuat untuk kandidat agent umum.
- **TIDAK mengubah prinsip fail-closed**: determinisme ≠ keamanan. Sandbox ketat (no-net, fs terbatas, audit, keluaran hanya usulan/diff) tetap wajib; validasi & eksekusi tetap di Go.
- Catatan: keandalan bergantung pada satu proyek milik sendiri (coupling provider↔produk) — di WO nanti, kebutuhan determinisme sebaiknya dinyatakan sebagai kontrak interface (bukan asumsi diam-diam), agar bisa diganti/simulasi bila donis-agent berubah.
