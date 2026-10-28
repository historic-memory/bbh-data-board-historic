---
title: 'WO-02: Dokumen resident — DELETE & PATCH'
description: H-7 — tambah service/handler/route untuk menghapus dan mengedit dokumen resident (parsial), ter-guard resident.write.
status: complete
created: "2026-10-28"
updated: "2026-10-28"
tags:
    - work-order
    - resident
    - backend
    - documents
---
# WO-02: Dokumen resident — DELETE & PATCH (H-7)

## Referensi
- SPEC: A-7..A-9, AC-5..AC-7, GAP-2.
- `plugins/resident/services/query.go:322` (`ListDocuments`), `:332` (`AddDocument`).
- `plugins/resident/handlers/handlers.go:267-305` (documents handlers + swagger annotations).
- `plugins/resident/plugin.go:74-75` (route).
- Tabel `resident_documents` (migration `..._create_resident_core_tables.up.sql`).

## Tugas
1. **`UpdateDocumentInput`** di `services/service.go` (parsial, semua pointer):
   ```go
   type UpdateDocumentInput struct {
       DocType *string `json:"doc_type"`
       URL     *string `json:"url"`
       Label   *string `json:"label"`
   }
   ```
2. **`Service.DeleteDocument(residentID, docID string) error`**:
   - `Delete` dengan syarat `id = docID AND resident_id = residentID`; `RowsAffected == 0` → `ErrNotFound`.
   - Hard delete (A-8).
3. **`Service.UpdateDocument(residentID, docID string, in UpdateDocumentInput) (*model.ResidentDocument, error)`**:
   - Ambil baris (`id`+`resident_id`); tidak ada → `ErrNotFound`.
   - `doc_type != nil` → `validateEnum` terhadap `ValidDocumentTypes`; `""` → `400`.
   - `url != nil` → trim; blank → `400`. 
   - `label != nil` → `nilIfBlank`.
   - `updated_at = now`; kembalikan baris terbaru.
4. **Handler** (`handlers.go`): `DeleteDocument` (204) & `PatchDocument` (200) + anotasi swagger
   (`@Router /api/admin/residents/{id}/documents/{docId} [delete|patch]`, `@Security BearerAuth`).
5. **Route** (`plugin.go`), urutan setelah `/:id/documents`:
   ```go
   residents.DELETE("/:id/documents/:docId", middleware.RequirePermission(authSvc, "resident.write"), h.DeleteDocument)
   residents.PATCH("/:id/documents/:docId", middleware.RequirePermission(authSvc, "resident.write"), h.PatchDocument)
   ```
6. Pastikan `:docId` tidak bentrok dengan route lain (gin tree) — taruh di grup `/:id/documents`.

## Test
- `service_db_test.go`: delete dokumen milik resident lain → `ErrNotFound`; delete sukses; patch label saja
  (url tetap); patch `url` blank → `ErrInvalid`; patch `doc_type` invalid → `ErrInvalid`.
- Handler test (opsional): status code 204/200/400/404.

## Kriteria Selesai
- AC-5, AC-6, AC-7 terpenuhi.
- `go build ./...` + `go test ./plugins/resident/...` lulus.
- Swagger memuat kedua endpoint setelah WO-04.

## Catatan Implementasi (2026-10-28)

- `UpdateDocumentInput` di `services/service.go` (semua pointer). `DeleteDocument` / `UpdateDocument`
  di `services/query.go`; handler `DeleteDocument` (204) & `PatchDocument` (200) di `handlers/handlers.go`;
  route di `plugin.go` (guard `resident.write`).
- Query selalu menyertakan `resident_id` → dokumen milik resident lain = `404` (bukan 403), tidak
  membocorkan kepemilikan.
- `doc_type` blank → `400` (bukan dianggap NULL), invalid → `400`; `url` blank/spasi → `400`;
  `label` blank → `NULL`; `updated_at` hanya disentuh bila ada field yang berubah.
- Verifikasi (container `app`): build + vet + `go test ./plugins/resident/...` lulus.
  Test DB baru `services/document_db_test.go` PASS (bukan skip): hard delete (baris benar-benar hilang),
  404 lintas resident, patch label saja, blank label → NULL, blank url/invalid doc_type ditolak tanpa
  mengubah baris.
- Verifikasi HTTP (port 8082, actor root): create 201 → PATCH label 200 → PATCH blank url 400
  (`url wajib`) → PATCH `doc_type` invalid 400 (`unknown enum value`) → PATCH valid 200 →
  DELETE 204 → DELETE ulang 404 → DELETE resident lain 404 → tanpa token 401.
- **Catatan:** jalur 403 (`resident.write` tanpa wildcard) BELUM teruji end-to-end — aktor uji root
  memegang `*` (H-8). Guard terpasang; verifikasi butuh admin non-root.
- Kode: `d6114a2`.
