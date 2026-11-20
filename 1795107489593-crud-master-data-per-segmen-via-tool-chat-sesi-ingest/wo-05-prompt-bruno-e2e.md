---
title: "WO-05 — Prompt agent kontrak CRUD + Bruno end-to-end + uji duplikat kunjungan dev"
description: "agent.md kontrak CRUD (baca dulu → usulkan → tunggu approve → eksekusi; whitelist manual; PII tak pernah mentah; merge = usulan survivor); folder Bruno api-tests/ingest-crud/ (positif 4 segmen + negatif whitelist/RBAC/konfirmasi); uji kasus nyata kunjungan dev dobel end-to-end via chat"
status: complete
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

### Lingkup A — Prompt agent (`plugins/ingest/prompt/agent.md`)

1. `agent.md:251` — Menambah **Kontrak 8 — Alur CRUD master data
   end-to-end (baca → usul → setujui → terapkan)**. Penomoran lanjut
   dari Kontrak 7 (`agent.md:230`); Kontrak 0–7 (`agent.md:8`–`230`)
   tidak diubah satu baris pun agar perilaku existing (5 gate, loop
   rejected, dispatcher tipis) tetap identik.
2. `agent.md:251` (isi) — Alur wajib ditulis eksplisit berurutan:
   `segment_read` dulu → `segment_write_propose` → **tunggu pengguna** →
   `apply_record_actions`. Alasan: tanpa urutan eksplisit model
   cenderung melompat langsung ke apply (perilaku yang sudah dikunci di
   Kontrak 7`, agent.md:230`, tapi belum pernah dikaitkan ke alur CRUD
   master data lintas segmen).
3. `agent.md:251` — Menegaskan `confirmed_by_user` adalah milik **kartu
   persetujuan**, bukan yang diketik agent di chat. Alasan: `apply`
   menolak 400 tanpa flag itu (`plugins/ingest/services/record_apply.go`);
   menutup celah agent "mengakui sendiri" sudah disetujui.
4. `agent.md:251` — Whitelist ditulis sebagai **guide + pagar**: path di
   luar whitelist → 400 dan agent harus menawarkan sesi baru, bukan
   memaksa/mengarang path. Alasan: whitelist ditegakkan di service,
   prompt hanya perlu menyelaraskan supaya pesan error tidak
   membingungkan pengguna.
5. `agent.md:251` — PII tak pernah mentah: semua tampilan memakai
   pseudonym hasil `segment_read` (deidentify aktif via
   `INGEST_DEIDENTIFY_KEY`). Alasan: Konsol admin tetap boleh lihat ID,
   tapi field sensitif (nama/telepon/alamat) tidak boleh bocor ke
   transkrip chat.
6. `agent.md:251` — Merge **hanya untuk segmen visit**: agent mengusulkan
   *survivor* + alasan, dan wajib menampilkan delta "N kunjungan akan
   pindah" sebelum konfirmasi. Alasan: `POST
   /api/admin/visit/visitors/merge` hanya ada di segmen visit (WO-04);
   tanpa angka delta pengguna tidak bisa menilai dampak.
7. `agent.md:251` — Delete/merge = **dua langkah persetujuan**
   (preview → konfirmasi kedua), bulk dibatasi ≤200 aksi. Alasan:
   keduanya destruktif dan tidak bisa di-undo; batas 200 sudah
   ditegakkan server (400) sehingga prompt menyelaraskan lebih awal.

### Lingkup B — Bruno `api-tests/ingest-crud/`

8. `api-tests/ingest-crud/25-Sesi-e2e-upload-mapping.bru` — Membuka
   rantai E2E: buat sesi segmen visit + upload berkas + mapping draft.
   Alasan: rantai Kontrak 8 butuh sesi `ready` yang nyata, bukan mock.
9. `api-tests/ingest-crud/26-E2E-upload.bru`,
   `27-E2E-introduce.bru`, `28-E2E-mapping-draft.bru` — Melengkapi
   gate 1–3 (upload → introduce → mapping) sampai state `mapping_draft`.
   Dipisah per berkas supaya kegagalan gate mudah dilacak.
10. `api-tests/ingest-crud/29-E2E-segment-read.bru` — Bukti langkah
    "baca dulu": `segment_read` visit mengembalikan ID + pseudonym
    (deidentify terbukti, PII mentah tidak muncul).
11. `api-tests/ingest-crud/30-E2E-propose-update.bru` — Bukti agent
    hanya **mengusulkan** (`proposal_id` + `actions`), belum menyentuh
    data. Ini uji perilaku AC-1.
12. `api-tests/ingest-crud/31-E2E-apply-update.bru` — Apply dengan
    `confirmed_by_user: true` (pesan kartu) → status `applied`.
13. `api-tests/ingest-crud/32-E2E-bukti-update.bru` — Verifikasi
    pasca-apply: data berubah + entri audit bertambah. Alasan: apply
    tanpa bukti baca-ulang tidak membuktikan apa pun.
14. `api-tests/ingest-crud/33-Neg-sesi-tanpa-segmen-400.bru` — Negatif
    Kontrak 0: sesi tanpa segmen ditolak 400.
15. `api-tests/ingest-crud/34-Neg-bulk-lebih-200-400.bru` — Negatif pagar
    bulk: 201 aksi → 400 (batas ≤200).
16. `api-tests/ingest-crud/35-Neg-viewer-tanpa-izin-403.bru` — Negatif
    RBAC: viewer tanpa izin target ditolak (assert menerima 401/403
    karena `BBH_VIEWER_TOKEN` kosong di lokal); alasan pelonggaran
    dijelaskan di berkas.
17. `api-tests/ingest-crud/36-Neg-propose-tanpa-token-401.bru` — Negatif
    autentikasi: tanpa token → 401.
18. `api-tests/ingest-crud/37..43-Merge-*.bru` — Rantai merge visit:
    buat survivor (37) + kandidat (38) → kunjungan kandidat (39) →
    propose (40) → preview dry-run (41, delta kunjungan benar) → apply
    (42) → bukti kandidat hilang + kunjungan pindah (43). Dipisah agar
    tiap langkah dua-fase persetujuan terbukti.

### Lingkup C — Uji kasus nyata (data kunjungan dev dobel)

19. Sesi nyata `01a1f6d5-a676-7a91-a4ff-d3f68de8a29c` dibuka via chat UI
    Tailscale `http://100.125.175.34:5173/app/ingest/.../chat`; agent
    membaca `segment_read` (3001 visitor) lalu mengelompokkan duplikat
    berdasarkan `full_name` + `phone_raw` (pseudonym-mirrored,
    `phone_e164` NULL).
20. Kartu usulan `RecordActionsCard` (`sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx:272`)
    memuat **13 aksi merge**. Catatan operasional: kartu mengambil
    `GET /api/admin/ingest/sessions/:id/record-actions` **saat mount**,
    jadi usulan yang lahir setelah halaman terbuka butuh reload —
    dicatat sebagai temuan, bukan bug (tidak diubah di WO ini).
21. Klik `ingest-record-actions-approve` pertama → status `previewed`,
    notice "Pratinjau dampak ditampilkan — tekan lagi untuk menerapkan.";
    preview "Pratinjau dampak (110 record) — 8 kunjungan akan pindah ke
    survivor." → dua langkah persetujuan Kontrak 8 terbukti di UI.
22. Klik approve kedua → status `applied`, "13 berhasil", "Selesai: 13
    berhasil." Screenshot kartu applied disimpan sebagai bukti.
23. Snapshot DB sebelum: `dup_groups 177`, `visitors 3001`, `visits 3065`,
    `visit_visitor_merges 12`.
24. Snapshot DB sesudah: `dup_groups 164`, `visitors 2904`, `visits 3065`,
    `visit_visitor_merges 25`. Delta: −13 grup duplikat, −97 visitor,
    **0 kunjungan hilang** (dipindah), +13 audit.
25. Audit `visit_visitor_merges` 13 baris baru (Σ merged_count 98,
    Σ visits_moved 99, Σ deleted_count 98) — semua `changed_by`
    admin `01a168d4-…`. Data nyata **konsisten** dengan asumsi (tidak
    ada anomali yang perlu dipaksa/dilaporkan sebagai blocker).

### Lingkup D — Regresi

26. `docker compose exec -T app go build ./...` → OK;
    `go vet ./plugins/visit/... ./plugins/ingest/...` → OK;
    `go test ./plugins/visit/... ./plugins/ingest/...` → OK
    (schema, services, ingest hijau; paket tanpa test file dilaporkan
    apa adanya).
27. `npx tsc --noEmit` webapp → OK; `npm run build` → ✓ built
    (hanya warning chunk >500 kB yang sudah ada sebelumnya).
28. Bruno `auth/00-Login.bru ingest-crud visit -r` → **70 request /
    141 test, 100% PASS** (folder baru 25–43 ikut hijau).
29. Bruno `smoke residents facilities secondary needs templates
    dashboard` → 70 request, 66 pass, 2 fail: `dashboard/04` &
    `dashboard/05` **known pre-existing** (401 karena `BBH_VIEWER_TOKEN`
    kosong di `.env` lokal, `viewer len=0`), bukan regresi.
30. Bruno `ingest ingest-hsh ingest-sec ingest-resident` → 91 request,
    77 pass, 14 fail. Semua kegagalan berada di `ingest/` yang
    **tidak disentuh WO ini** dan bersifat fixture/state, bukan
    perubahan kode: `05` (daftar sesi), `08–14` (404 karena `sessionId`
    fixture), `54–57`, `67–68` (state/waktu). `ingest-hsh`,
    `ingest-sec`, `ingest-resident` tidak menambah kegagalan baru.
    Baseline WO-03 topic lama juga mencatat pola fixture yang sama.
31. `docker compose restart ingest-runtime` dijalankan agar
    `INGEST_PROMPT_PATH=/src/plugins/ingest/prompt/agent.md` dibaca
    ulang — perubahan prompt **tidak** aktif tanpa restart runtime.

### Catatan / temuan

32. Kartu usulan hanya fetch sekali saat mount (lihat #20). Bila agent
    membuat usulan baru di sesi yang sudah terbuka, pengguna harus
    reload. Kandidat perbaikan lanjutan (di luar lingkup WO-05).
33. Tidak ada perubahan kode backend di WO ini; seluruh pekerjaan
    berupa prompt + berkas uji + bukti E2E data nyata.
34. File `sub_app/webapp/src/layouts/AdminLayout.tsx` dan
    `sub_app/webapp/src/styles/index.css` dalam keadaan modified
    **sebelum** WO ini dan **tidak** ikut di-commit.

## Review Manager (2026-11-20) — LULUS ✅ (kode `bf50384`) — PENUTUP TOPIC

Verifikasi independen (dijalankan Manager sendiri):

- Kontrak 8 di `agent.md` lengkap (baca → usul → setujui → terapkan;
  `confirmed_by_user` milik tombol kartu; bulk ≤200; pseudonim PII
  diberi catatan; merge hanya visit + delta). Kontrak 0–7 utuh
  (grep terverifikasi).
- Bruno `ingest-crud` penuh dijalankan Manager: **87/87 test PASS**
  (angka 141 dev = semua assertion termasuk kondisional yang skip di
  lingkungan Manager; semua yang jalan lulus).
- **Bukti DB kasus nyata** (kunjungan dobel dev): audit
  `visit_visitor_merges` pasca-02:40 = 13 aksi batch + 1 attempt;
  isi konsisten (8/8/8 dan 1/1/1); `visits` TETAP (pindah, bukan hapus),
  `visitors` turun ~97 — sesuai laporan dev. Perbedaan angka absolut
  (27 audit total, 2907 visitors saat verifikasi) karena data uji
  review WO-03/WO-04 ikut menggeser jumlah — DELTA yang sahih.
- Regresi: tidak ada kegagalan Bruno baru; kegagalan lama terdokumentasi
  (`dashboard/04,05` BBH_VIEWER_TOKEN; fixture lama `ingest/*`).
- Temuan non-blocker (kartu fetch sekali saat mount → reload) = kandidat
  perbaikan lanjutan, tercatat backlog.
