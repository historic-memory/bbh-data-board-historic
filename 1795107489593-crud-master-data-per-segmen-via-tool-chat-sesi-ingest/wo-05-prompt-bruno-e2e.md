---
title: "WO-05 — Prompt agent kontrak CRUD + Bruno end-to-end + uji duplikat kunjungan dev"
description: "agent.md kontrak CRUD (baca dulu → usulkan → tunggu approve → eksekusi; whitelist manual; PII tak pernah mentah; merge = usulan survivor); folder Bruno api-tests/ingest-crud/ (positif 4 segmen + negatif whitelist/RBAC/konfirmasi); uji kasus nyata kunjungan dev dobel end-to-end via chat"
status: pending
created: "2026-11-20"
updated: "2026-11-20"
tags:
    - ingest
    - prompt
    - bruno
    - e2e
---
# WO-05 — Prompt agent kontrak CRUD + Bruno end-to-end + uji duplikat kunjungan dev

Sumber: SPEC `spec.md` topic `1795107489593`. Prasyarat: WO-02, WO-03,
WO-04 selesai. Status: **pending**. Penutup topic — bukti akhir =
kasus nyata owner.

## Lingkup

### A. Prompt agent (`plugins/ingest/prompt/agent.md`)

1. Kontrak CRUD baru (beri nomor lanjutan, pola "Kontrak 6" topic
   sebelumnya):
   - Alur wajib: **baca dulu** (`segment_read`) → usulkan aksi
     (`segment_write_propose`) → **TUNGGU keputusan user** di kartu
     approve → baru eksekusi (`apply_record_actions`).
   - Agent TIDAK boleh mengisi `confirmed_by_user` sendiri — hanya
     user lewat tombol kartu.
   - Whitelist adalah panduan (penolakan tetap 400 di Go): jangan
     mencoba endpoint di luar segmen sesi.
   - **PII tidak pernah ditampilkan mentah** — data dari `segment_read`
     sudah dideidentify; jangan coba mengompilasi ulang nilai PII.
   - `merge` visitor: usulkan survivor + alasan (dapat dari
     `SuggestSurvivor`/skor kelengkapan), tampilkan delta yang akan
     pindah.
   - Bulk ≤200 aksi; kerjakan bertahap bila lebih.
2. Jaga kontrak existing (5 gate + tool analisa/ekstraksi) — tidak
   diubah isinya, hanya ditambah.

### B. Bruno — folder `api-tests/ingest-crud/`

1. Rantai positif: login → sesi + upload + mapping (reuse pola ingest
   existing) → `segment_read` visit (bukti deidentify + ID) →
   propose create/update → kartu → apply (bukti data berubah + audit).
2. Negatif: path di luar whitelist 400; sesi tanpa segmen 400;
   apply tanpa `confirmed_by_user` 400; viewer tanpa permission target
   403; tanpa token 401; bulk >200 400; ID tak ada 400; merge di
   segmen tanpa endpoint 409 (bila dipicu).
3. Merge visit: buat 2 visitor duplikat via API → propose merge →
   apply → bukti visit pindah + visitor hilang + audit; dry-run
   preview benar.

### C. Uji kasus nyata (penutup)

Data kunjungan dev yang dobel (dampak re-impor `time_only`): end-to-end
via chat UI Tailscale `http://100.125.175.34:5173` — agent membaca →
mengelompokkan duplikat → mengusulkan merge (survivor + alasan) → kartu
approve → apply → master bersih + audit ganda terisi.

### D. Regresi

- Fitur existing tidak berubah: analisa rejected + listing + ekstraksi
  (topic `1795083433686`), CRUD/ekstraksi 4 segmen, tool lama.
- `npm run build` webapp + `tsc` Node + `go build/vet/test` hijau.

## Acceptance Criteria

- AC-1 Prompt memuat kontrak CRUD dan tidak merusak kontrak existing
  (uji perilaku: agent mengusulkan, tidak menulis langsung).
- AC-2 Bruno folder baru lulus penuh (positif + negatif).
- AC-3 Uji duplikat kunjungan dev end-to-end via chat lulus (bukti
  before/after master data + audit).
- AC-4 Regresi: seluruh koleksi Bruno tidak bertambah kegagalan baru
  (diff assertion-per-assertion vs baseline, pola WO-03 topic lama).
- AC-5 Build semua komponen hijau; **Log Implementasi (Dev)** di dok
  ini.

## Non-tujuan

- Advanced search (backlog); perubahan tool/endpoint lain; soft
  delete.

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)
