---
title: "Step 13 — Auto-Wakeup Fix Plan"
type: modification-plan
status: draft
tags: [paperclip, shannon, auto-wakeup, debate-router, step-13, planning]
date: 2026-05-10
---

# Step 13 — Auto-Wakeup Fix (Debate Router)

> **Masalah:** Setelah `debate-router.ts` merouting issue ke agen berikutnya (ubah assignee), agen yang baru ditugaskan tidak otomatis mulai bekerja. Paperclip hanya memberikan wakeup trigger ke agen ketika ada **comment baru** masuk di issue mereka — bukan saat assignment berubah. Akibatnya human harus posting comment manual untuk "memancing" agen melanjutkan.
>
> **Fix:** Setiap kali `debate-router.ts` menyelesaikan routing, server otomatis posting sebuah system comment ke issue yang berisi instruksi untuk agen berikutnya. Comment ini yang menjadi wakeup trigger.

---

## Codebase yang Perlu Dibaca Pertama

Sebelum mulai coding, baca file-file ini di VPS (path: `/home/paperclip/paperclip-shannon`):

```
server/src/services/debate-router.ts         ← logic routing utama
server/src/routes/issues.ts                  ← comment handler, tempat routing dipanggil
server/src/routes/issues.ts (baris NEXT_COMMAND) ← lihat parseNextCommand() sebagai referensi
packages/db/src/schema/issue-comments.ts     ← schema comment (untuk buat system comment)
packages/db/src/schema/issues.ts             ← lihat kolom assignee + pipeline_stage
server/src/services/                         ← cari apakah ada helper "postComment" atau "createComment"
```

**Tujuan membaca:**
1. Pahami return value dari routing function di `debate-router.ts` — apa yang dikembalikan setelah routing? (next agent ID, next stage, dll)
2. Pahami di mana tepatnya routing dipanggil di `routes/issues.ts` — setelah DB update assignee, di situlah kita insert system comment
3. Cari apakah sudah ada helper function untuk insert comment secara programatik — kalau ada, tinggal pakai; kalau tidak, kita buat inline

---

## Root Cause Detail

```
Comment dari agen masuk
        ↓
routes/issues.ts menerima POST /issues/:id/comments
        ↓
debate-router.ts parsing signal tag → tentukan next agent
        ↓
DB update: issues.assignee_agent_id = nextAgentId
           issues.pipeline_stage = nextStage  (kalau advance)
        ↓
Response 200 OK — SELESAI
        ↓
[GAP] Agen baru tidak tahu dia diassign. Tidak ada wakeup.
        ↓
Human harus posting comment manual → baru agen bangun
```

Yang kita tambahkan di antara "DB update" dan "Response 200 OK":

```
        ↓
DB insert: issue_comments row baru dengan:
  - issue_id: issue yang sedang diproses
  - author_type: "system"  (atau "agent" dengan agent_id = nextAgentId)
  - body: pesan instruksi ke agen berikutnya
  - created_at: now()
        ↓
Paperclip melihat comment baru → kirim wakeup ke agen yang assigned
        ↓
Agen bangun dan mulai bekerja
```

---

## Desain System Comment per Routing Path

Setiap routing path membutuhkan pesan yang berbeda. Agen harus tahu **kenapa** dia diassign dan **apa yang harus dia lakukan** — tanpa perlu baca seluruh thread dari awal.

### Path 1: CEO → PM (setelah [TASK])
```
[System — Pipeline]
CEO telah mem-framing task ini. PM Lead, kamu sekarang ditugaskan untuk membuat PM Brief.

Baca [TASK] dari CEO di atas, lalu post [PM BRIEF v1] dengan format:
  Stories, Scope (in/out), Dependencies, Estimate, Open questions for SWE.
```

### Path 2: PM → SWE Lead (setelah [PM BRIEF v1] atau [PM BRIEF v2])
```
[System — Pipeline]
PM Lead telah posting [PM BRIEF v{n}]. SWE Lead, giliranmu untuk review.

Baca [TASK] CEO dan [PM BRIEF v{n}] di atas, lalu post:
  - [CONFIRMED] + ≥2 verification questions (jika brief oke secara teknis)
  - [CONCERNS] + ≥2 enumerated concerns (jika ada masalah teknis)
```

### Path 3: SWE Lead → PM (setelah [CONCERNS], untuk revisi)
```
[System — Pipeline]
SWE Lead punya concerns teknis. PM Lead, ini revisi ke-{n} (maks 2).

Baca [CONCERNS] SWE Lead di atas dan post [PM BRIEF v{n+1}] yang sudah direvisi.
Jika setelah 2 revisi SWE Lead masih concerns, CEO akan mengambil keputusan.
```

### Path 4: SWE Lead / PM → CEO (eskalasi setelah 2 revisi)
```
[System — Pipeline]
Debate PM ↔ SWE Lead telah mencapai batas 2 revisi tanpa konsensus.
CEO, dibutuhkan keputusan kamu.

Baca seluruh thread [PM BRIEF] dan [CONCERNS] di atas, lalu post [DECISION: <resolusi>].
Jika kamu butuh klarifikasi dari human, gunakan request_confirmation.
```

### Path 5: CEO/SWE [CONFIRMED] → UI/UX (jika NEEDS_DESIGN: yes)
```
[System — Pipeline]
Spec telah dikonfirmasi. UI/UX Lead, kamu ditugaskan karena issue ini butuh design scope.

Baca [TASK] CEO dan [PM BRIEF] final di atas, lalu post [UI/UX SPEC] dengan:
  Component inventory, layout notes, interaction notes.
```

### Path 6: UI/UX → SWE Lead (setelah [UI/UX SPEC])
```
[System — Pipeline]
UI/UX Lead telah selesai. SWE Lead, review [UI/UX SPEC] di atas dan post [LGTM] jika siap dispatch workers.
```

### Path 7: SWE Lead confirm tanpa design (NEEDS_DESIGN: no)
```
[System — Pipeline]
Spec dikonfirmasi, tidak ada design scope. SWE Lead, post [LGTM] untuk memulai implementation.
```

### Path 8: [LGTM] → advance ke design/implementation
```
[System — Pipeline]
[LGTM] diterima. Pipeline stage sekarang: {nextStage}.
SWE Lead akan mulai dispatch workers untuk implementasi.
```

---

## Implementasi: Lokasi Persis di Codebase

### Langkah 1 — Baca return type dari debate-router

Di `server/src/services/debate-router.ts`, cari fungsi utama routing. Kemungkinan bentuknya:

```typescript
// Yang sudah ada (perkiraan):
export function routeDebateSignal(params: {
  issueId: string
  signal: DebateSignal        // [TASK], [PM BRIEF], [CONFIRMED], dll
  currentAssigneeId: string
  agents: Agent[]
  revisionCount: number
  needsDesign: boolean
}): RoutingResult

type RoutingResult = {
  nextAgentId: string | null
  nextStage: PipelineStage | null
  action: 'route' | 'escalate' | 'human_checkpoint' | 'advance'
  reason: string
}
```

Kita perlu tambahkan ke `RoutingResult`:
```typescript
type RoutingResult = {
  nextAgentId: string | null
  nextStage: PipelineStage | null
  action: 'route' | 'escalate' | 'human_checkpoint' | 'advance'
  reason: string
  systemComment: string | null   // ← TAMBAH INI
}
```

Setiap path routing sudah menghasilkan `RoutingResult` — kita tambahkan field `systemComment` dengan teks yang sesuai per path.

### Langkah 2 — Insert system comment di routes/issues.ts

Di `server/src/routes/issues.ts`, cari blok setelah `debate-router` dipanggil dan DB diupdate:

```typescript
// Yang sudah ada (perkiraan):
const routing = routeDebateSignal({ ... })

if (routing.nextAgentId) {
  await db.update(issues)
    .set({ assigneeAgentId: routing.nextAgentId })
    .where(eq(issues.id, issueId))
}

if (routing.nextStage) {
  await db.update(issues)
    .set({ pipelineStage: routing.nextStage })
    .where(eq(issues.id, issueId))
}

// ← TAMBAHKAN DI SINI:
if (routing.systemComment) {
  await db.insert(issueComments).values({
    id: createId(),
    issueId: issueId,
    authorType: 'system',
    authorAgentId: null,       // system, bukan agent
    body: routing.systemComment,
    createdAt: new Date(),
  })
}
```

**Catatan:** Cek exact column names di `packages/db/src/schema/issue-comments.ts` — sesuaikan field names di atas.

### Langkah 3 — Verifikasi: apakah Paperclip trigger wakeup dari system comment?

Ini yang perlu diverifikasi dulu sebelum coding:

1. Buka `server/src/routes/issues.ts` — cari bagian setelah insert comment yang mengirim notifikasi/wakeup ke agen
2. Cek apakah ada SSE (Server-Sent Events) atau WebSocket push yang dikirim ke agen
3. Kalau wakeup hanya dikirim untuk comment dari `authorType: 'agent'` atau `'user'` — kita perlu pastikan `'system'` juga ter-trigger, atau gunakan `authorType: 'agent'` dengan `authorAgentId` = previous agent

**Fallback jika system comment tidak trigger wakeup:**
Set `authorAgentId = currentAgentId` (agen yang baru selesai) dengan `authorType: 'agent'` — ini pasti memicu wakeup karena Paperclip sudah handle agent comments.

---

## Perubahan di Agent Instructions (AGENTS.md)

Selain server fix, setiap agen perlu instruksi eksplisit di `AGENTS.md`-nya:

```markdown
## Autonomous Continuation

Kamu akan diassign ke issue ini bersama sebuah [System — Pipeline] comment yang menjelaskan apa yang harus kamu lakukan. Baca comment tersebut, baca thread discussion board di atas, lalu langsung mulai bekerja tanpa menunggu input tambahan dari human.

Jika kamu butuh klarifikasi yang tidak bisa kamu resolve sendiri:
- Post pertanyaanmu sebagai bagian dari signal tag yang sesuai (contoh: "Open questions for SWE: ..." di [PM BRIEF])
- Gunakan request_confirmation HANYA jika pipeline tidak bisa lanjut tanpa jawaban human
```

Ini sebagai **fallback** — bahkan jika wakeup sudah bekerja, instruksi ini memastikan agen tahu harus apa saat bangun.

---

## Urutan Pengerjaan di VPS

```
1. Baca debate-router.ts → pahami RoutingResult type yang ada sekarang
2. Baca routes/issues.ts → pahami di mana routing dipanggil dan hasil DB update
3. Baca packages/db/src/schema/issue-comments.ts → catat exact column names
4. Cari mekanisme wakeup: apakah ada notif/push setelah comment insert?
5. Tambah field systemComment ke RoutingResult di debate-router.ts
6. Isi systemComment per routing path (teks dari seksi "Desain System Comment" di atas)
7. Tambah insert system comment di routes/issues.ts setelah DB update
8. Update AGENTS.md tiap agen (CEO, PM, SWE Lead, UI/UX Lead) dengan instruksi autonomous continuation
9. Typecheck: pnpm -r typecheck
10. Test: buat test issue di shannon-v3-test, walk through debate, verifikasi auto-wakeup
```

---

## Acceptance Criteria

- [ ] Setelah CEO post [TASK], PM otomatis mendapat system comment dan mulai posting [PM BRIEF] tanpa human intervensi
- [ ] Setelah PM post [PM BRIEF], SWE Lead otomatis bangun dan post [CONFIRMED] atau [CONCERNS]
- [ ] Setelah SWE Lead post [CONCERNS], PM otomatis bangun untuk revisi
- [ ] Setelah 2x concerns, CEO otomatis diassign dan mendapat system comment untuk eskalasi
- [ ] Setelah [LGTM], pipeline_stage otomatis advance (sudah ada di v3 — verifikasi tetap jalan)
- [ ] Kalau agen benar-benar butuh info human → request_confirmation muncul dan human bisa jawab
- [ ] v2 company (shannon-factory-2) tidak terpengaruh — system comment hanya muncul di issue dengan debate signal tags
- [ ] pnpm -r typecheck exit 0
- [ ] Unit test debate-router.test.ts tetap pass (22 test) + tambah test untuk systemComment field

---

## Yang Tidak Berubah di Step 13

- Logic routing debate-router.ts (siapa ke siapa) → tidak diubah
- Validation gate ([CONFIRMED] wajib ≥2 items) → tidak diubah
- NEXT_COMMAND: mechanism untuk v2 → tidak diubah
- Worker dispatch → tidak diubah
- Pipeline stages setelah spec → tidak diubah

---

## Setelah Step 13 Selesai → Step 14: URS Skills

Setelah auto-wakeup fix verified, lanjut ke port URS-first skills:
- `foundation--urs-ingest` (CEO skill)
- `foundation--sprint-plan` (PM skill)
- `urs--create-issues` (PM skill, Shannon-specific)
- `foundation--shape-spec` tambah `--from-urs` path

Lihat [[Step 13 — URS Skills Plan]] (akan dibuat setelah Step 13 selesai).

---

## Related

- [[Shannon Company Templates — v2 vs v3]] — konteks v2 vs v3
- [[Roundtable Discussion Architecture]] — spec MAD Asimetris yang diimplementasi
- [[Paperclip Shannon — Modification Plan Phase 2]] — rencana Step 12 (roundtable)
- [[Paperclip Shannon — Current Condition 2026-05-04]] — baseline sebelum Step 13
- [[VPS-Shannon/paperclip-dev-setup]] — cara akses VPS dan jalankan dev server
