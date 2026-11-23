---
title: notes
description: "Kronologi keputusan & langkah berikutnya topic chatbot ACP."
status: progress
created: "2026-11-24"
updated: "2026-11-24"
tags: [notes]
---

# Catatan

## Kronologi
- 2026-11-23: riset ACP terbentuk (topic 1795469702082) — dua makna ACP dijernihkan.
- 2026-11-24 (pagi): owner mengusulkan agent umum ala OpenCode sebagai manipulator dokumen ingest ("1 paket"). Tercatat sebagai alternatif desain di topic sandbox 1795414203062 (save 477bbe1) + donis-agent deterministik (save 07d7dbf).
- 2026-11-24: owner memutuskan: (a) hapus/clear "fitur deterministik" dari bbh-data-board; (b) buat chatbot BARU terkoordinasi ACP tanpa menyentuh ingest chat eksisting; (c) dulu DICATAT saja, lalu riset alternatif agent selain OpenCode.
- Audit pembersihan dilakukan Manager sebelum eksekusi: fitur "deterministik" praktis tidak ada di repo (classifier/Jev tak pernah dikodekan; teks deterministik milik provider donis-agent). Yang tak boleh ikut terhapus: deidentify HMAC (PII) + pipeline deterministik fail-closed. Pembersihan cukup mencabut komentar/referensi usang — DIUNDUR ke housekeeping topic ini.

## Langkah berikutnya
1. **Riset alternatif agent selain OpenCode** (permintaan owner) — kandidat: Claude Code (ACP adapter), Gemini CLI, agent dari Agent Stack (BeeAI), dsb. Bandingkan: lisensi, kualitas tool, keamanan (perission model), ukuran, integrasi LLM (donis-agent?), dukungan ACP asli.
2. Setelah riset: kunci O-1..O-5 di spec, lalu SPEC final + WO bertahap (pola checkpoint).
3. Housekeeping: cabut komentar/referensi deterministik usang (rules.go:18 + dok topic ingest) — masuk WO housekeeping topic ini.
