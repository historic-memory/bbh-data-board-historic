---
title: "WO-CP1: Container Goose terisolasi + smoke test ACP"
description: "Bangun service container `goose-agent` (goose binary + `goose serve`/`goose acp`) dengan isolasi ketat (jaringan internal, non-root, resource cap, scratch read-only) dan smoke test ACP end-to-end dari Node (initialize → session → prompt → permission deny)."
status: planned
created: "2026-11-24"
updated: "2026-11-24"
tags:
    - chatbot
    - goose
    - acp
    - docker
    - sandbox
    - checkpoint-cp1
---
# WO-CP1: Container Goose terisolasi + smoke test ACP

## Latar belakang

Topic ini (chatbot baru terkoordinasi ACP) sudah final di level desain (save `354875e`): agent = **Goose** (O-1), container **terpisah** (O-2), provider LLM = **donis-agent** OpenAI-compatible (O-3). WO ini adalah **CP-1** (checkpoint pertama): infrastruktur agent hidup TANPA menyentuh data bisnis — semua test memakai fixture fiktif.

Fakta teknis dari riset dok resmi (2026-11-24, goose-docs.ai — AAIF/Agentic AI Foundation):

1. **Transport**:
   - `goose acp` = mode subprocess (JSON-RPC over stdio) — contoh resmi memakai Node: `spawn("goose", ["acp"], { stdio: ["pipe","pipe","inherit"] })`.
   - `goose serve` = mode server HTTP/WebSocket resmi: `GOOSE_SERVER__SECRET_KEY='<long-random>' goose serve` → endpoint default `http://127.0.0.1:3284/acp` (`ws://` untuk WS), auth via header `X-Secret-Key` (atau `?token=` untuk browser WS); opsi origin allowlist ada di `goose-cli-commands#serve-options` + guide `remote-goose-server`.
   - **Karena O-2 = container terpisah, transport yang direkomendasikan: `goose serve` HTTP di jaringan internal Docker**. Alternatif `docker exec -i … goose acp` (stdio bridge) DITOLAK karena membutuhkan mount `docker.sock` ke container Node = eskalasi privilege yang tidak bisa diterima.
   - Catatan jujur: transport remote standar ACP (spec) masih WIP; `goose serve` adalah mekanisme goose-native yang stabil hari ini — kedua ujung (goose server + client Node kita) kita kontrol sendiri, jadi risiko rendah.

2. **Peringatan resmi yang memvalidasi desain sandbox**: "Permission callbacks are not a sandbox: goose's configured mode determines when approval is requested, and its tools run with the backend's access to the machine." → kartu permission ACP adalah UX-gate, BUKAN sandbox. Sandbox sesungguhnya = container + filesystem + jaringan. Ini alasan isolasi 6 lapis tidak bisa dinegosiasikan.

3. **Provider OpenAI-compatible (jalur donis-agent)** — dua cara resmi:
   - Env langsung pada provider bawaan `openai`: `GOOSE_PROVIDER=openai`, `GOOSE_MODEL=<model>`, `OPENAI_API_KEY=<key>`, `OPENAI_HOST=<endpoint>` (default `api.openai.com`), `OPENAI_BASE_PATH=<path>` (default `v1/chat/completions`), opsi `OPENAI_CUSTOM_HEADERS`.
   - **Custom provider JSON** (direkomendasikan): file JSON dengan `name`, `engine: "openai"`, `display_name`, `base_url`, `models[]` (name, context_limit, input/output_token_cost), `requires_auth` + `api_key_env`. Shareable/bisa di-commit (tanpa secret). Via `goose configure` atau taruh file di direktori konfigurasi.
   - **Prasyarat model**: goose sangat bergantung **tool calling**; model tanpa tool-calling hanya bisa chat completion dan semua extension goose harus dimatikan. Pastikan model donis-agent yang dipakai mendukung tool/function calling.

4. **SDK ACP client (Node)**: `@agentclientprotocol/sdk@1.5.0` (`npm install @agentclientprotocol/sdk@1.5.0`) — API: `ndJsonStream` (stdio), `client({name})`, `connectWith`, `initialize {protocolVersion: acp.PROTOCOL_VERSION, clientCapabilities}`, `agent/session/new {cwd, mcpServers:[]}`, `agent/session/prompt {sessionId, prompt:[{type:"text",text}]}`, notifikasi `session/update` (`agent_message_chunk`, `tool_call`, `tool_call_update`), request `session/requestPermission` (jawab `{outcome:{outcome:"cancelled"}}` untuk menolak). Untuk transport `goose serve`, SDK menyediakan transport HTTP/WS setara — gunakan itu, bukan ndJsonStream stdio.

## Tujuan

1. Service compose baru **`goose-agent`** hidup dan sehat, menjalankan goose dalam container yang terisolasi ketat.
2. Smoke test ACP end-to-end lulus dari sisi Node: handshake → session → prompt balasan → permission deny terbukti memblokir aksi tool (fail-closed terverifikasi).
3. Tidak ada perubahan backend Go / UI — murni infra + script smoke.

## Lingkungan kerja (scope)

- **Baru** (boleh dibuat executor):
  - `Dockerfile.goose` — image goose (opsi A: build dari Dockerfile resmi repo `aaif-goose/goose` (docs BUILDING_DOCKER); opsi B: base slim + unduh release binary pinned `GOOSE_VERSION=v1.60.2` (atau terbaru stabil saat eksekusi) dari GitHub releases). Pilih yang stabil; **pin versi** agar reproducible.
  - `docker-compose.yml` — tambah service `goose-agent` + network internal `goose-net` (lihat rancangan di bawah).
  - `sub_app/ingest_runtime/scripts/acp-smoke.mjs` — script smoke test (Node ≥22, ESM).
  - `.env.example` — tambah entri placeholder: `GOOSE_SERVER__SECRET_KEY=`, `GOOSE_PROVIDER=`, `GOOSE_MODEL=`, `OPENAI_API_KEY=`, `OPENAI_HOST=`, `OPENAI_BASE_PATH=` (nilai nyata di `.env`, TIDAK di-commit).
- **DILARANG disentuh**: semua kode Go (`internal/`, `plugins/`), UI (`sub_app/webapp`), runtime CopilotKit existing (`sub_app/ingest_runtime` selain menambah file script baru di folder `scripts/`), file ingest chat eksisting.

## Rancangan service `goose-agent` (docker-compose)

```yaml
  goose-agent:
    build: { context: ., dockerfile: Dockerfile.goose }
    restart: unless-stopped
    user: "1000:1000"                      # non-root
    env_file: .env
    environment:
      GOOSE_PROVIDER: ${GOOSE_PROVIDER:-openai}
      GOOSE_MODEL: ${GOOSE_MODEL}
      OPENAI_HOST: ${OPENAI_HOST}          # endpoint donis-agent
      OPENAI_BASE_PATH: ${OPENAI_BASE_PATH:-v1/chat/completions}
      GOOSE_SERVER__SECRET_KEY: ${GOOSE_SERVER__SECRET_KEY}
    command: ["goose", "serve", "--port", "3284"]
    volumes:
      - goose-home:/home/goose/.config     # config/logs goose (writable)
      - goose-scratch:/work                # ruang kerja per-sesi (CP-3 di-mount snapshot read-only)
    working_dir: /work
    mem_limit: 1g
    cpus: 1.0
    pids_limit: 100
    networks: [goose-net]                  # BUKAN network `backend`
    ports: []                              # tidak expose ke host; akses hanya dari network internal
    healthcheck:
      test: ["CMD", "curl", "-s", "-o", "/dev/null", "http://127.0.0.1:3284/acp"]
      interval: 10s
      timeout: 3s
      retries: 5
networks:
  goose-net:
    internal: true                          # tanpa egress default; provider LLM diarahkan via proxy/egress khusus
```

Catatan isolasi jaringan (lapis 2): jika endpoint provider LLM (donis-agent) berada DI LUAR host, `internal: true` memutus egress. Dua pilihan valid — pilih dan catat di notes:
- (a) `internal: false` + dua network (goose-net internal untuk akses Node; network egress yang difirewall hanya ke host provider) — lebih sederhana, kewajiban firewall di host (ufw/iptables: DROP semua, ALLOW hanya IP provider).
- (b) tambah sidecar proxy egress kecil (squid/envoy allowlist) antara goose dan provider — lebih ketat, opsional.

## Isi script smoke `acp-smoke.mjs` (perilaku wajib)

1. Baca `GOOSE_URL`, `GOOSE_SERVER__SECRET_KEY` dari env; buat koneksi ACP ke `http://goose-agent:3284/acp` (WS) via SDK.
2. `initialize` dengan `acp.PROTOCOL_VERSION`; tolak eksplisit bila versi tidak sama.
3. `session/new` dengan `cwd: "/work"`, `mcpServers: []`.
4. **Test A (round-trip)**: prompt `"Say hello in one sentence."` → kumpulkan `agent_message_chunk` text → prompt request harus selesai dengan `stopReason` yang menandakan turn selesai (`end_turn`). Print semua event.
5. **Test B (permission deny / fail-closed)**: prompt yang memancing aksi tool, mis. `"Create a file named probe.txt in the working directory with content 'x'."` → saat `session/requestPermission` datang, jawab `{outcome:{outcome:"cancelled"}}` → setelah turn selesai, verifikasi **`/work/probe.txt` TIDAK ada** (via exec kedua atau baca dir lewat prompt berikutnya "list files" dan cek respons). Bila file TERBENTUK = FAIL.
6. **Test C (permission approve jalur minimal, opsional bila mode default meminta izin)**: ulangi Test B tapi setujui → file terbentuk → hapus lagi. Ini membuktikan jalur approve juga berfungsi.
7. Print `[SMOKE] baseURL=...` di awal (pola kewajiban test) dan exit code 0 bila semua lulus, 1 bila gagal.

Cara jalankan: `docker compose exec ingest-runtime node scripts/acp-smoke.mjs` (ingest-runtime di-join ke `goose-net` untuk CP-1) atau dari host bila port sementara dipublish di dev.

## Hardening tambahan (wajib dicatat di notes saat eksekusi)

- Matikan semua extension bawaan goose via `config.yaml` (developer/shell/memory/computercontroller dll.) — whitelist kosong untuk CP-1; enable selektif hanya di CP-2/CP-3 bila perlu. Ref: docs `using-extensions#enablingdisabling-extensions` + `config-files`.
- Mode goose (approval mode) diset ke mode paling konservatif yang tersedia; kartu permission ACP tetap jadi gate UX (bukan sandbox).
- `GOOSE_SERVER__SECRET_KEY` dibuat `openssl rand -hex 32` oleh user; TIDAK boleh muncul di log atau di-commit.
- Log goose (stderr/`/home/goose/.config`) tidak boleh memuat isi prompt bisnis — untuk CP-1 aman karena fixture fiktif; sikap ini dikunci untuk CP-2/CP-3.

## Kriteria selesai

1. `docker compose up -d --build goose-agent` sukses; `docker compose ps` status healthy; container berjalan sebagai non-root (verifikasi `docker compose exec goose-agent id`).
2. Isolasi terverifikasi: `goose-agent` TIDAK berada di network `backend`; dari dalam container, koneksi ke `postgres:5432`, `keydb:6379`, dan API Go GAGAL (connection refused/unreachable). Catat hasil probe di notes.
3. Egress: hanya endpoint provider LLM yang bisa dijangkau (sesuai pilihan (a)/(b) di atas); akses ke internet umum GAGAL.
4. Smoke test lulus: Test A (round-trip balasan dari donis-agent), Test B (deny → file tidak terbentuk), Test C opsional (approve → file terbentuk lalu dibersihkan). Output script lengkap ditempel di notes topic.
5. Provider = donis-agent via custom provider JSON atau `OPENAI_HOST` env (bukan api.openai.com); identitas model terlihat di log smoke.
6. Tidak ada perubahan pada kode Go, UI, atau jalur ingest chat eksisting (verifikasi `git status` hanya menampilkan file baru yang diperbolehkan + `.env.example`).
7. Secrets tidak ter-commit (`.env` di gitignore; custom provider JSON tidak memuat nilai secret).

## Catatan teknis / risiko

- **Tool calling**: bila model donis-agent belum mendukung function/tool calling, Test B/C tidak berarti (goose tidak akan pernah minta izin tool) — WO ini tetap bisa lulus Test A saja, tapi CATAT dengan jujur sebagai limitasi dan eskalasi ke owner (pilih model tool-calling).
- `OPENAI_BASE_PATH` harus tepat: 404 biasanya berarti path salah (doc LiteLLM); 401 berarti key tidak termuat.
- goose menulis config ke `$HOME/.config` — volume `goose-home` wajib agar read-only rootfs tetap mungkin di iterasi lanjut.
- Versi goose: pin di Dockerfile (`GOOSE_VERSION` / tag rilis) supaya bump versi adalah keputusan sadar, bukan drift.
- Bila `goose serve` menuntut flag berbeda dari `--port` (cek `goose serve --help` di versi yang dipin; ada opsi host/origin di `goose-cli-commands#serve-options`), sesuaikan command dan catat di notes — jangan hardcode asumsi.
- Semua eksplorasi CLI (versi, help, config path) dilakukan di dalam container, tidak menginstal goose di host.
