---
title: "Desain — flow perbaikan baris rejected & tool call Node↔Go"
description: "Flow paten 5 tahap (report → perbaiki → evaluasi ulang → confirm → extract) + desain 3 tool call baru (get_rejected_rows, propose_row_fix, apply_fixes) mengikuti pola tool strict existing; keputusan P-1..P-6 + T-1..T-4"
status: draft
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - gate
    - ai-chat
    - desain
---
# Desain — Flow perbaikan baris rejected & tool call Node↔Go

Topic turunan `1795083433686` (extends `1793711295286`). Status: **draft —
menunggu keputusan owner** sebelum SPEC/WO dibuat. Semua rujukan kode
terverifikasi dari workspace 2026-11-19.

## 1. Flow paten (5 tahap)

```
[extract gagal gate / ada rejected]
  (1) REJECTED REPORT  — baca EnforcementReport sesi terakhir
                        (RowNo + Violates + Reason; dari rules.go:68)
  (2) PERBAIKI — user memilih salah satu jalur:
        a. RULE pengecualian → allow/skip (agent usulkan dari KATALOG
           exceptions.yaml; user approve; StatedBy=admin)
        b. NILAI baris → BERKAS PERBAIKAN BARU (v2) di-upload MENG GANTIKAN
           berkas sesi — bukan edit DB staging (P-1)
  (3) EVALUASI ULANG — enforcement jalan lagi atas berkas/rules baru;
        dry run WAJIB dulu (P-4)
  (4) CONFIRM ULANG — mapping_draft → mapping_confirmed
        (gate tidak boleh dilompati; revise existing dipakai)
  (5) EXTRACT ULANG → extracted (state machine existing; TIDAK ada state baru)
```

### Keputusan desain P-1..P-6 (usulan Manager, menunggu owner)

- **P-1 Perbaikan nilai = berkas baru, bukan edit DB.**
  Faktanya `latestUploadData` (`services/extraction.go`) selalu membaca
  berkas ASLI dari storage; staging (`*_import_batches/rows`) hanya jejak
  ekstraksi. Maka jalur nilai yang jujur: agent usulkan baris perbaikan →
  user approve → Go menghasilkan **CSV perbaikan** → di-upload menggantikan
  berkas sesi (invarian satu-berkas/sesi + mekanisme replace sudah ada).
  Edit baris di staging DB = DITOLAK.
- **P-2 Dua jalur, dua tempat keputusan:**
  (a) rule `allow`/`skip` → disimpan di profil (rules append-only, O-20);
  (b) nilai baris → berkas perbaikan di storage. Profil tak tersentuh jalur (b).
- **P-3 AI TIDAK pernah menulis — hanya mengusulkan.** Semua tulisan lewat
  action API Go (RBAC + audit, D-11); rule selalu `StatedBy: user`;
  auto-apply oleh sistem DITOLAK.
- **P-4 Dry run wajib** pada evaluasi ulang sebelum extract sungguhan.
- **P-5 Konteks ke chat = deidentified.** Daftar rejected yang dikirim ke
  LLM lewat jalur `BuildSafePayload`/deidentify — PII rejected tidak pernah
  mentah ke model (pola CP-2: safe-payload di depan LLM).
- **P-6 Tanpa state baru, tanpa tabel baru.** Flow hidup di state machine
  existing (`mapping_draft`/`revise`/`confirm`/`extract`). Tambahan data
  hanya: endpoint baca laporan rejected per sesi (dari `EnforcementReport`
  yang sudah ada; staging juga bisa jadi sumber audit).

## 2. Desain tool call Node ↔ Go

Pola yang diikuti = tool strict existing (`handlers/agent_tools.go` +
`sub_app/ingest_runtime/copilot-runtime.ts`):
- **Go = pemilik fungsi**: endpoint `/admin/ingest/tools/*` ber-guard
  `ingest.write` + audit `provider='ingest-tool'`; semua kebijakan
  (katalog rule, fail-closed, deidentify) dieksekusi di Go.
- **Node = pendaftar skema**: JSON Schema → `convertJsonSchemaToZodSchema`
  (tanpa dependensi baru), `BuiltInAgent` per-request, tool memanggil API
  Go via **Bearer milik user**; Node hanya meneruskan tool-call model —
  tanpa logika bisnis (D-11).

### Tool baru (3)

**T-1 `get_rejected_rows`** — baca, tidak menulis.
- Endpoint Go: `POST /admin/ingest/tools/get_rejected_rows`
- Input: `session_id`, `max_rows` (dipotong backend).
- Output: `EnforcementReport` terakhir (skipped+rejected per `RowNo`,
  `Violates`, `Reason`) + nilai baris terkait dalam bentuk AMAN
  (deidentify, pola sample ≤N baris) + katalog `exceptions.yaml` segmen
  yang relevan agar agent bisa mengusulkan rule yang SAH.
- Sumber data: report ekstraksi terakhir + (audit) staging rows.

**T-2 `propose_row_fix`** — mengusulkan, BELUM menerapkan.
- Endpoint Go: `POST /admin/ingest/tools/propose_row_fix`
- Input: `session_id`, `fixes[]` di mana tiap fix salah satu dari:
  `{kind:"rule", violates, action:"allow"|"skip", scope, note}` (rule_id
  HARUS ada di katalog → `ErrUnknownRule` bila tidak) atau
  `{kind:"row", row_no, values:{field: nilai_baru}}`.
- Perilaku: disimpan sebagai **proposal** (belum diterapkan ke profil maupun
  berkas) + ter-audit. Agent tidak bisa lompat ke penerapan.
- Output: `proposal_id` + ringkasan + pesan bahwa penerapan butuh
  persetujuan user.

**T-3 `apply_fixes`** — penerapan, dipicu USER (bukan inisiatif agent).
- Endpoint Go: `POST /admin/ingest/tools/apply_fixes`
- Input: `session_id`, `proposal_id`, `confirmed_by_user: true` (wajib
  eksplisit — flag hilang → 400 fail-closed).
- Perilaku per kind:
  - `rule` → validasi ulang terhadap katalog (`RuleSet.Validate`) → tambah
    ke rules profil dengan `StatedBy=<admin>`, append-only versi baru.
  - `row` → Go bentuk **CSV perbaikan** dari berkas asli + nilai baru →
    disimpan sebagai berkas perbaikan → menggantikan berkas sesi
    (mekanisme replace existing).
- Lanjutan: state kembali `mapping_draft` (revise path) → user confirm
  ulang → dry run → extract (flow §1 tahap 3-5).

### Aturan yang TIDAK berubah (paten bawaan)

1. Agent tidak pernah menulis master data/profil langsung — semua lewat
   tool = API Go (RBAC + audit).
2. Rule hanya dari katalog `exceptions.yaml` segmen — `RuleSet.Validate`
   sudah menolak rule sintetis (fail-closed, gratis dari mesin existing).
3. Konteks ke LLM selalu terdeidentify (P-5).
4. Node tidak pernah menyentuh DB — hanya skema + forward.
5. Dry run wajib di evaluasi ulang (P-4).

## Terbuka (untuk owner)

- O-21: ukuran maksimal proposal per sesi (batas jumlah baris yang boleh
  diperbaiki lewat chat sebaiknya dibatasi — usulan: ringkas per rule id,
  bukan per baris).
- O-22: siapa yang boleh approve (`ingest.write` cukup, atau perlu
  permission baru?) — usulan: cukup `ingest.write`, sama dengan flow
  editor.
- SPEC + pecah WO disusun setelah P-1..P-6/T-1..T-4 disetujui.
