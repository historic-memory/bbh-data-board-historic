---
title: "WO-CP1: Runner Goose terisolasi + launcher spawn dinamis (config dari DB)"
description: "Bangun container runner `goose-agent` (goose binary + launcher) dengan isolasi ketat TANPA secret statik; launcher meng-spawn `goose serve` per-sesi dengan konfigurasi provider yang disusun dari payload (jalur DB di CP-2); smoke test 2 fase (hardening + spawn dinamis + ACP deny)."
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
# WO-CP1: Runner Goose terisolasi + launcher spawn dinamis (revisi 2)

> **REVISI 2 (2026-11-24)** — keputusan owner: sandbox Goose harus **dinamis** — di-spin saat load, dengan start-config yang disusun dari data aplikasi (config provider yang tersimpan di DB). Versi lama ("`goose serve` statik dengan env compose") DIBATALKAN karena env compose bersifat statik: dibekukan saat container dibuat, ikut ter-restart dengan nilai lama, dan 1 config untuk semua sesi. Arsitektur baru: **runner persistent + launcher spawn per-sesi**.

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

5. **Konfigurasi dinamis dari DB (grounding kode existing)**: infra provider SUDAH ADA — `plugins/ingest/services/providers.go` (CRUD provider + `ActiveProviderConfig()` di baris 286) dan `plugins/ingest/services/provider_key.go` (key tersimpan terenkripsi AES-GCM di DB; dekripsi HANYA Go via env `INGEST_DB_KEY_ENC`; prinsip D-11: Node tidak pernah memegang secret mentah). Alur secret untuk sandbox dinamis: **DB → dekripsi di Go → payload spawn internal → env proses goose** — secret tidak pernah menyentuh compose/`.env`/frontend/log.

## Tujuan

1. Service compose baru **`goose-agent`** (runner) hidup dan sehat: image goose + launcher, isolasi ketat, **TANPA secret statik di compose/env**.
2. Launcher berfungsi: `POST /spawn` menghidupkan goose per-sesi dengan konfigurasi dinamis dari payload; `POST /kill` mematikan + wipe; `GET /healthz`.
3. Smoke test 2 fase lulus: (fase 1) hardening & isolasi statik; (fase 2) spawn dinamis dengan 2 config fixture BERBEDA membuktikan konfigurasi per-sesi benar-benar terpisah, plus Test ACP A/B/D (round-trip, permission deny fail-closed, isolasi konfigurasi antar sesi).
4. Tidak ada perubahan pada kode Go existing / UI — murni file baru + script smoke (integrasi `ActiveProviderConfig()` dari Go masuk WO CP-2).

## Lingkungan kerja (scope)

- **Baru** (boleh dibuat executor):
  - `Dockerfile.goose` — image runner (opsi A: build dari Dockerfile resmi repo `aaif-goose/goose`, docs BUILDING_DOCKER; opsi B: base slim + unduh release binary pinned `GOOSE_VERSION=v1.60.2`/terbaru stabil dari GitHub releases). **Pin versi**. Selain goose, install launcher binary ke image.
  - `cmd/goose-launcher/main.go` (+ file package pendukung di `cmd/goose-launcher/`) — launcher kecil: HTTP listen internal, endpoint `/spawn`, `/kill`, `/healthz` (kontrak di bawah). Kode BARU di dalam modul Go existing, tanpa dependensi eksternal baru.
  - `docker-compose.yml` — tambah service `goose-agent` + network internal `goose-net` + join `app` (Go) ke `goose-net` (dual-homing, lihat catatan).
  - `sub_app/ingest_runtime/scripts/acp-smoke.mjs` — smoke 2 fase (Node ≥22, ESM).
  - `.env.example` — tambah placeholder NON-secret: `GOOSE_LAUNCHER_ADDR=:9315`, `GOOSE_SESSION_MAX_MINUTES=30`, `GOOSE_MAX_SESSIONS=5`. **Tidak ada lagi env provider/secret statik** (GOOSE_PROVIDER/OPENAI_API_KEY/GOOSE_SERVER__SECRET_KEY dihapus dari desain compose).
- **DILARANG disentuh**: semua kode Go existing (`internal/`, `plugins/`, `cmd/server`, `cmd/console`), UI (`sub_app/webapp`), runtime CopilotKit existing (`sub_app/ingest_runtime` selain menambah file di `scripts/`), file ingest chat eksisting. CP-1 TIDAK memodifikasi Go existing — smoke memakai payload fixture; koneksi launcher ↔ `ActiveProviderConfig()` masuk WO CP-2.

## Rancangan service `goose-agent` (runner) + kontrak launcher

```yaml
  goose-agent:
    build: { context: ., dockerfile: Dockerfile.goose }
    restart: unless-stopped
    user: "1000:1000"                      # non-root
    environment:                           # TIDAK ADA secret/provider di sini
      GOOSE_LAUNCHER_ADDR: ":9315"
      GOOSE_STATE_ROOT: /home/goose        # launcher buat HOME per-sesi di sini
      GOOSE_WORK_ROOT: /work
      GOOSE_SESSION_MAX_MINUTES: ${GOOSE_SESSION_MAX_MINUTES:-30}
      GOOSE_MAX_SESSIONS: ${GOOSE_MAX_SESSIONS:-5}
    command: ["/usr/local/bin/goose-launcher"]
    volumes:
      - goose-state:/home/goose            # HOME per-sesi (dibuat & di-wipe launcher)
      - goose-scratch:/work                # ruang kerja; CP-3 nanti mount snapshot read-only
    working_dir: /work
    mem_limit: 1g
    cpus: 1.0
    pids_limit: 200                        # sisi atas: sesi aktif × (goose + child)
    networks: [goose-net]                  # BUKAN network `backend`
    ports: []                              # tidak expose ke host
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://127.0.0.1:9315/healthz"]
      interval: 10s
      timeout: 3s
      retries: 5
networks:
  goose-net:
    internal: true
```

Container `app` (Go) ditambah network `goose-net` (**dual-homing**): tetap di `backend` untuk postgres/keydb, plus `goose-net` khusus memanggil launcher. `goose-agent` TIDAK ikut `backend`, jadi dari dalam runner, postgres/keydb/API Go tetap TIDAK bisa dijangkau. Catatan egress (provider LLM): bila endpoint donis-agent DI LUAR host, `internal: true` memutus egress — pilih dan catat: (a) `internal: false` + firewall host (ALLOW hanya IP provider); (b) sidecar proxy egress allowlist. Untuk donis-agent yang jalan di VPS/host yang sama, jaringan internal tetap cukup.

### Kontrak launcher (wajib)

- `POST /spawn` — body: `{ "config": { "provider": "openai", "model": "…", "openai_host": "…", "openai_base_path": "v1/chat/completions", "openai_api_key": "…" } }` → launcher:
  1. Validasi payload; tolak bila field wajib kosong (fail-closed; pesan error TIDAK mengandung nilai secret).
  2. Buat dir per-sesi: `HOME=/home/goose/sessions/<id>` + `cwd=/work/<id>`; `<id>` = random hex 16.
  3. Tulis `config.yaml` di HOME sesi (mode approval konservatif, SEMUA extension dimatikan, sesuaikan docs `using-extensions#enablingdisabling-extensions` + `config-files`) dan siapkan env proses: `GOOSE_PROVIDER`, `GOOSE_MODEL`, `OPENAI_HOST`, `OPENAI_BASE_PATH`, `OPENAI_API_KEY` (dari payload), `GOOSE_SERVER__SECRET_KEY` = random hex 32 per-sesi, plus var port `goose serve` yang valid di versi dipin (verifikasi via `goose serve --help`; pola `GOOSE_SERVER__*` terlihat dari doc resmi).
  4. Spawn `goose serve` per-sesi dengan env proses di atas — **secret via env proses, BUKAN argv** (argv terbaca via `/proc`), BUKAN env global launcher.
  5. Respons `{ "session": "<id>", "port": <p>, "secret": "<hex>", "expires_at": "…" }`.
  6. Timeout: kill otomatis saat `GOOSE_SESSION_MAX_MINUTES` habis; cap jumlah sesi aktif `GOOSE_MAX_SESSIONS` → 429 bila penuh.
- `POST /kill` — body `{ "session": "<id>" }` → kill proses (beserta anak), hapus rekursif HOME sesi + `/work/<id>`, bebaskan port. Idempoten.
- `GET /healthz` → `{ "ok": true, "sessions_active": N }`.
- Endpoint hanya melayani network internal (tanpa auth ekstra di CP-1; token internal opsional di CP-2).
- Audit minimal ke stderr launcher: waktu, session id, provider, model, status spawn/kill, durasi — **tanpa secret**.

Catatan isolasi jaringan tetap berlaku dari versi lama: hanya endpoint provider LLM yang boleh terjangkau dari runner (lihat pilihan (a)/(b) di atas); akses ke internet umum GAGAL.

## Script smoke `acp-smoke.mjs` (2 fase — perilaku wajib)

**Fase 1 — hardening & isolasi statik (TANPA memanggil goose):**
1. Print `[SMOKE] baseURL=http://goose-agent:9315` (pola kewajiban test).
2. `docker compose exec goose-agent id` → non-root; `docker inspect goose-agent --format '{{.Config.Env}}'` → TIDAK memuat `OPENAI_API_KEY`/secret provider.
3. Probe dari dalam runner: koneksi ke `postgres:5432`, `keydb:6379`, API Go → semua GAGAL; `GET /healthz` launcher dari `app` → BERHASIL.
4. Probe egress: endpoint provider DONIS-AGENT bisa dijangkau (sesuai pilihan (a)/(b)); domain internet umum → GAGAL.

**Fase 2 — spawn dinamis + ACP (SDK `@agentclientprotocol/sdk@1.5.0`, transport HTTP/WS):**
5. **Test A (config dinamis #1 = donis-agent)**: `POST /spawn` dengan fixture config donis-agent (dibaca dari env smoke) → dapat `{session, port, secret}` → konek ACP ke `http://goose-agent:<port>/acp` (auth `X-Secret-Key`) → `initialize` dengan `acp.PROTOCOL_VERSION` (tolak eksplisit bila versi beda) → `session/new {cwd:"/work/<id>", mcpServers:[]}` → prompt `"Say hello in one sentence."` → kumpulkan `agent_message_chunk` (text) → prompt request selesai dengan `stopReason` turn selesai.
6. **Test B (permission deny / fail-closed)**: di sesi yang sama, prompt `"Create a file named probe.txt in the working directory with content 'x'."` → saat `session/requestPermission` datang jawab `{outcome:{outcome:"cancelled"}}` → setelah turn selesai verifikasi `/work/<id>/probe.txt` TIDAK ada (`docker compose exec goose-agent test ! -f /work/<id>/probe.txt`, plus prompt lanjutan "list files in the working directory" untuk cek silang). File TERBENTUK = FAIL.
7. **Test C (approve path, opsional bila mode default meminta izin)**: ulangi B dengan menyetujui → file terbentuk → `POST /kill` membersihkan.
8. **Test D (bukti konfigurasi per-sesi — inti revisi 2)**: `POST /spawn` kedua dengan config fixture BERBEDA yang sengaja salah (`openai_host` unreachable) → prompt sesi #2 GAGAL dengan error koneksi provider, sementara sesi #1 tetap sehat → bukti env/config TIDAK bocor antar sesi → `POST /kill` kedua-duanya → verifikasi `/home/goose/sessions/*` dan `/work/*` kosong.
9. Exit 0 bila semua wajib lulus, 1 bila gagal; seluruh output ditempel di notes topic.

Cara jalankan: `docker compose exec ingest-runtime node scripts/acp-smoke.mjs` (ingest-runtime ikut di-join ke `goose-net` untuk CP-1) — atau dari host bila launcher dipublish sementara di dev.

## Hardening tambahan (wajib dicatat di notes saat eksekusi)

- Matikan semua extension bawaan goose via `config.yaml` (developer/shell/memory/computercontroller dll.) — whitelist kosong untuk CP-1; enable selektif hanya di CP-2/CP-3 bila perlu. Ref: docs `using-extensions#enablingdisabling-extensions` + `config-files`.
- Mode goose (approval mode) diset ke mode paling konservatif yang tersedia; kartu permission ACP tetap jadi gate UX (bukan sandbox).
- `GOOSE_SERVER__SECRET_KEY` per-sesi dibuat launcher (random hex 32) — TIDAK lagi statik dari `.env`; TIDAK boleh muncul di log atau di-commit.
- Log goose (stderr/`/home/goose/.config`) tidak boleh memuat isi prompt bisnis — untuk CP-1 aman karena fixture fiktif; sikap ini dikunci untuk CP-2/CP-3.

## Kriteria selesai

1. `docker compose up -d --build goose-agent` sukses; `docker compose ps` status healthy; `docker compose exec goose-agent id` = non-root.
2. `docker inspect goose-agent --format '{{.Config.Env}}'` TIDAK memuat secret provider (`OPENAI_API_KEY` dst.) — env hanya konfig launcher non-secret.
3. Isolasi: `goose-agent` TIDAK di network `backend`; probe postgres/keydb/API Go dari dalam runner GAGAL; `GET /healthz` launcher dari `app` BERHASIL (dual-homing bekerja).
4. Smoke fase 2 lulus: Test A (balasan dari donis-agent via config payload), Test B (deny → file tidak terbentuk), Test D (dua config berbeda terisolasi; kill + wipe bersih). Output lengkap ditempel di notes topic.
5. Provider = donis-agent diarahkan dari payload spawn (bukan compose); identitas model terlihat di log smoke.
6. Tidak ada perubahan kode existing: `git status` hanya menampilkan file baru yang diperbolehkan (`Dockerfile.goose`, `cmd/goose-launcher/*`, edit `docker-compose.yml`, `scripts/acp-smoke.mjs`, `.env.example`).
7. Secret tidak ter-commit dan TIDAK pernah muncul di log/audit launcher maupun output smoke.

## Catatan teknis / risiko

- **Tool calling**: bila model donis-agent belum mendukung function/tool calling, Test B/C tidak berarti (goose tidak akan pernah minta izin tool) — WO ini tetap bisa lulus Test A saja, tapi CATAT dengan jujur sebagai limitasi dan eskalasi ke owner (pilih model tool-calling).
- `OPENAI_BASE_PATH` harus tepat: 404 biasanya berarti path salah (doc LiteLLM); 401 berarti key tidak termuat.
- goose menulis config ke `$HOME/.config` — launcher membuat HOME per-sesi di volume `goose-state`, jadi rootfs image bisa read-only di iterasi lanjut.
- Implementasi launcher: Go standard library saja (`net/http`, `os/exec`) — paket BARU dalam modul, tanpa dependensi eksternal baru; secret hanya lewat env proses, JANGAN argv (terbaca via `/proc`).
- Nama env `goose serve` untuk port (pola `GOOSE_SERVER__*` terlihat dari `GOOSE_SERVER__SECRET_KEY` di docs) diverifikasi via `goose serve --help` di versi yang dipin — jangan hardcode asumsi.
- Versi goose: pin di Dockerfile (`GOOSE_VERSION` / tag rilis) supaya bump versi adalah keputusan sadar, bukan drift.
- Bila `goose serve` menuntut flag berbeda dari asumsi port (cek `goose serve --help`; ada opsi host/origin di `goose-cli-commands#serve-options`), sesuaikan dan catat di notes.
- Semua eksplorasi CLI (versi, help, config path) dilakukan di dalam container, tidak menginstal goose di host.
