---
title: WO-01c — Eksperimen MCP + housekeeping (CP-1, sub 3/3)
description: "Sub-checkpoint CP-1: eksperimen MCP di luar jalur utama, catat temuan, rapikan dokumen & keputusan lanjut sebelum CP-2."
status: progress
created: "2026-11-03"
---
# WO-01c — Eksperimen MCP + Housekeeping (Sub-Checkpoint CP-1, 3/3)

Induk: `wo-01-setup-copilotkit-sandbox.md` (dipecah 2026-11-04). Urutan: 1a → 1b
→ **1c** (terakhir di CP-1). **Prasyarat: WO-01b lulus AC-nya.**

## Tujuan

Menutup CP-1: eksperimen MCP (keputusan tetap **eksploratif**, tidak jadi standar
arsitektur), merekam semua temuan belajar, dan merapikan dokumen sebelum masuk
CP-2 (real case `visit`).

## Lingkup

### 1. Eksperimen MCP (di luar jalur utama)

- Pasang SATU MCP server kecil ke CopilotKit di sandbox; catat di notes topic:
  - kompatibilitas dengan proxy OpenAI-compatible + CopilotKit runtime;
  - effort setup vs tool backend biasa (WO-01b) — perbandingan nyata;
  - risiko (jalur auth, audit, guard) bila MCP dipakai serius.
- **Eksperimen TIDAK menjadi standar**; keputusan apakah MCP dipakai di CP-2+
  direkam sebagai keputusan di notes (default: tidak — tool backend saja).

### 2. Housekeeping penutup CP-1

- Rekaman effort riil adaptasi CopilotKit di proxy (dari WO-01a) + pola tool (dari
  WO-01b) — masukan langsung untuk rancangan CP-4.
- Rapikan swagger & Bruno untuk semua endpoint sandbox; bersihkan kode eksperimen
  yang gagal/tidak dipakai (biarkan yang hidup).
- Checklist readiness CP-2 di notes: env `INGEST_LLM_*` aman, permission seed
  benar, tidak ada data bisnis yang pernah lewat sandbox.
- Perbarui status WO-01a/b/c + CP-1 di master index (`work-order.md`) dengan bukti.

## Acceptance Criteria

- AC-7 Temuan eksperimen MCP tercatat di notes topic (kompatibilitas, effort, risiko).
- AC-8 Keputusan lanjut soal MCP terekam; `go vet`/`go test` hijau; `npm run build`
  hijau; dokumen & status checkpoint diperbarui.

## Non-tujuan

- Semua hal CP-2 (sesi strict, state machine, deidentify nyata, ekstraksi `visit`)
  — baru mulai setelah CP-1 ditutup lewat checkpoint ini.

## Risiko

- Jangan habiskan waktu berlebihan di MCP — batasi eksperimen ini ke satu server
  kecil + catatan; nilai utamanya adalah KEPUTUSAN yang terekam, bukan fiturnya.

---

## Implementasi (Dev Agent, 2026-11-04)

### Bukti AC-7 — temuan MCP tercatat

| Item | Bukti |
| --- | --- |
| Satu server MCP kecil, di luar jalur utama | `sub_app/webapp/runtime/mcp-experiment/server.ts` (±90 LOC, tool `mcp_echo`), tidak ada di `docker-compose`, dijalankan manual lalu dimatikan |
| Tool benar-benar dipakai | run `POST /api/copilotkit/agent/default/run` menghasilkan `TOOL_CALL_START/ARGS/END` + `TOOL_CALL_RESULT` = `mcp-pong: halo-mcp`, lalu `RUN_FINISHED` |
| Kompatibilitas | didukung native `BuiltInAgent.mcpServers` (`@modelcontextprotocol/sdk@1.32.2` sudah ada — **tanpa dependency baru**); aman terhadap shim `provider-compat.ts`; kalau server MCP mati, chat tetap jalan (tool MCP di-skip) |
| Effort vs tool backend | tabel perbandingan di `notes.md` — MCP ±1 jam untuk echo, tetapi guard/audit/lifecycle harus dibangun sendiri; tool Go WO-01b sudah membawa guard+audit gratis |
| Risiko | panggilan tool MCP **tidak** menghasilkan baris `ingest_audit` dan tidak melewati RBAC Go (tool jalan penuh di Node) → bertentangan dengan D-11 |

Catatan tambahan: `Server` MCP tidak boleh dipakai lintas transport (crash
"Already connected to a transport") — pada mode stateless server harus dibuat
per request. Terverifikasi saat eksperimen.

### Bukti AC-8 — keputusan + kualitas

- **Keputusan: MCP TIDAK dipakai** di CP-2+ (default ditegakkan). Integrasi MCP
  di runtime hanya opt-in via `INGEST_MCP_EXPERIMENT_URL` (compose mengunci `""`),
  jadi tidak ada jalur MCP di jalur utama. Alasan lengkap di `notes.md`.
- Housekeeping selesai:
  - Proxy Go `/api/admin/ingest/llm-proxy/*` **dihapus** (keputusan owner):
    handler `LLMProxy`, `injectDefaultModel`, `Config`/`Configured`, konstanta
    `EnvBaseURL/EnvAPIKey/EnvModel` ikut dibuang. Verifikasi: endpoint → **404**,
    `/api/copilotkit/info` tetap **401** (guard hidup), swagger diregenerasi tanpa
    path proxy.
  - Telemetri: `COPILOTKIT_TELEMETRY_DISABLED="true"` di `docker-compose.yml`
    (service `ingest-runtime`).
  - Rekaman effort adaptasi CopilotKit (masukan CP-4) + checklist readiness CP-2
    ditulis di `notes.md`.
  - Bruno `api-tests/ingest/` dilengkapi: `04-Sesi-simpan-pesan`, `05-Sesi-daftar`,
    `06-Sesi-daftar-pesan`, `07-Audit-event` (sebelumnya hanya tool + upload).
- Gate kualitas: `go build ./...` OK, `go vet ./plugins/ingest/...` OK,
  `go test ./plugins/ingest/...` hijau, `npx tsc --noEmit` hijau, `npm run build`
  hijau (built in 20.82s), swagger teregenerasi.

### File tersentuh

- Hapus proxy: `plugins/ingest/{plugin.go,handlers/handlers.go,services/service.go,model/models.go}`
- Eksperimen MCP: `sub_app/webapp/runtime/mcp-experiment/server.ts` (baru),
  `sub_app/webapp/runtime/copilot-runtime.ts` (opt-in `mcpServers`)
- Konfigurasi: `docker-compose.yml` (komentar D-11 + `INGEST_MCP_EXPERIMENT_URL: ""`)
- Uji manual: `api-tests/ingest/04..07`
- Dokumen: `notes.md`, `work-order.md`, dokumen WO ini
