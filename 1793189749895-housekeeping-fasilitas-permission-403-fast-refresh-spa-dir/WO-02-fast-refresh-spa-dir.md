---
title: "WO-02: Fast Refresh & SPA_DIR"
description: "Pisahkan useAuth dari AuthContext.tsx (H-2) dan tambahkan env SPA_DIR pada plugins/pages dengan fallback + log error eksplisit (GAP-14)."
status: pending
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - frontend
  - spa
  - housekeeping
---

# WO-02: Fast Refresh & SPA_DIR (H-2, GAP-14)

## Referensi
- SPEC: A-5, A-6, AC-6, AC-7, GAP-3, GAP-4.
- `sub_app/webapp/src/auth/AuthContext.tsx` (mengekspor `AuthProvider` + `useAuth`).
- `plugins/pages/plugin.go:20` (`const spaDir = "./sub_app/webapp/dist"`).

## Tugas A — Fast Refresh (H-2)
1. Buat `src/auth/authContext.ts` berisi hanya `const AuthContext = createContext<...>(...)` (+ tipe).
2. Buat `src/auth/useAuth.ts` berisi hook `useAuth()`.
3. `AuthContext.tsx` hanya mengekspor komponen `AuthProvider`.
4. Perbarui semua import (`grep -r "useAuth" src`) — tidak boleh ada import rusak.
5. **Tidak** mengubah logika auth (status/user/permissions/login/logout/refresh).

## Tugas B — `SPA_DIR` (GAP-14)
1. Di `plugins/pages/plugin.go`, ganti konstanta hard-code menjadi fungsi:
   ```go
   // spaDirFromEnv: SPA_DIR menimpa default (relatif CWD).
   const defaultSPADir = "./sub_app/webapp/dist"
   ```
   - `dir := os.Getenv("SPA_DIR")`; kosong → `defaultSPADir`.
2. Saat `RegisterRoutes`, **verifikasi** keberadaan `index.html`:
   - ada → log `[pages] SPA dir: <abs path>`.
   - tidak ada → `log.Printf("[pages] WARNING: index.html tidak ditemukan di %q — /app akan 404", ...)`
     (jangan panik; server tetap jalan, tapi **jangan** senyap).
3. Pastikan tidak ada perubahan perilaku saat `SPA_DIR` kosong.

## Bukti
- Edit `AuthContext.tsx` saat `npm run dev` → HMR tanpa full reload (screenshot/log konsol).
- Jalankan server dari `/tmp` dengan `SPA_DIR=/abs/path/dist` → `/app/` 200.
- Tanpa `SPA_DIR` dari root repo → perilaku identik seperti sebelumnya.

## Kriteria Selesai
- AC-6, AC-7.
- `npm run build` + `tsc -b` lulus; `go build ./...` lulus.
