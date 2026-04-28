---
title: "Project — tanah-yang-dijanjikan"
type: project-log
tags: [project, land-management, geospatial, next-js]
status: discovering
---

# tanah-yang-dijanjikan

Land management system with geospatial capabilities.

---

## Tech Stack
- Next.js 16.1.6 + React 19
- Supabase (cloud, MCP enabled)
- TailwindCSS 4 + Shadcn/UI
- Leaflet + react-leaflet + Turf.js (geospatial/GIS)
- shpjs (shapefile import)
- TanStack Table, Recharts, date-fns

## Existing Features
| Route | Feature |
|---|---|
| `/pemetaan` | GIS / map view |
| `/peminjaman` | Lending / borrowing |
| `/permohonan` | Permit applications |
| `/petugas` | Personnel management |
| `/presensi` | Attendance |
| `/inventaris` | Inventory |
| `/monitoring` | Monitoring dashboard |
| `/upload-shp` | Shapefile upload |
| `/cekplot`, `/jabatan`, `/profil` | Plot check, position, profile |

---

## WhatsApp Automation Integration (2026-04-26)

### Goal
POC: kirim pesan WA otomatis dari sistem (notifikasi permohonan, dll) menggunakan **go-whatsapp-web-multidevice** (GoWA) sebagai backend WA gateway.

### Arsitektur
```
tanah-yang-dijanjikan (Next.js)
  └── lib/whatsapp/wa-client.ts   ← wrapper REST API
        └── fetch → GoWA REST API (localhost:3000)
                      └── whatsmeow → WhatsApp Web protocol
```

### File yang Dibuat / Dimodifikasi
| File | Keterangan |
|---|---|
| `lib/whatsapp/wa-client.ts` | Client wrapper: `checkStatus`, `sendTextMessage`, `sendFile`. Retry logic, auth header, phone formatter (08xxx → 628xxx@s.whatsapp.net) |
| `lib/whatsapp/types.ts` | Types: `WAApiResponse`, `WASendResult` |
| `test-wa.ts` | Script test manual via `npx tsx test-wa.ts` |
| `.env` | Env vars untuk test script (dotenv load `.env`, bukan `.env.local`) |

### Env Vars yang Dibutuhkan
```env
# Di .env (untuk test-wa.ts)
# Di .env.local (untuk Next.js app runtime)
WHATSAPP_API_URL=http://localhost:3000
WHATSAPP_BASIC_AUTH=user1:pass1   # sesuaikan dengan APP_BASIC_AUTH di GoWA
WHATSAPP_ENABLED=true
```

### Issue yang Ditemukan & Solusi

**Issue 1 — Auth 401 Unauthorized**
- Root cause: `wa-client.ts` fallback ke `admin:admin`, tapi GoWA pakai `user1:pass1`
- GoWA Basic Auth dikontrol di `src/.env` → `APP_BASIC_AUTH=user1:pass1,user2:pass2`
- `test-wa.ts` pakai `dotenv config()` yang load `.env` (bukan `.env.local`)
- Fix: buat `.env` di root Next.js project dengan `WHATSAPP_BASIC_AUTH=user1:pass1`
- Alternatif: kosongkan `APP_BASIC_AUTH=` di GoWA untuk dev/POC

**Issue 2 — `'use server'` di wa-client.ts**
- File punya directive `'use server'` → hanya bisa dipanggil dari Server Actions / API Routes
- Jangan import langsung di client component

### GoWA Setup (project terpisah)
- Repo: `D:\Side-Mission\project-1\wa-automation\go-whatsapp-web-multidevice`
- Run: `cd src && go run . rest` → port 3000
- Config: `src/.env` (copy dari `.env.example`)
- **Wajib scan QR** lewat `http://localhost:3000` sebelum bisa kirim pesan
- Panduan lengkap: `PANDUAN.md` di root repo GoWA

### Next Steps WA Integration
- [ ] Scan QR / login GoWA agar status `isConnected: true`
- [ ] Jalankan `npx tsx test-wa.ts` untuk verify end-to-end
- [ ] Integrasikan `sendTextMessage` ke flow permohonan (trigger saat status berubah)
- [ ] Pertimbangkan deploy GoWA di VPS Shannon untuk produksi

---

## Claude Code Tooling — MCP Obsidian

> Catatan ini berdiri sendiri — tidak terkait fitur aplikasi.

**Status: ✅ Verified working (2026-04-26)**

MCP Obsidian terkonfigurasi secara **global** di Claude Code. Semua sesi ke depannya bisa baca/tulis vault ini langsung tanpa akses file manual.

### Komponen yang Dikonfigurasi
| Komponen | Detail |
|---|---|
| **Plugin Obsidian** | MCP Tools by Jack Steam |
| **Binary** | `..\.obsidian\plugins\mcp-tools\bin\mcp-server.exe` |
| **Protocol** | HTTP port 27123 via Local REST API plugin |
| **Config lokasi** | `~/.claude.json` → `mcpServers.obsidian` (scope: user global) |
| **Auth** | `OBSIDIAN_API_KEY` di env section `~/.claude.json` |

### Troubleshooting yang Dilalui
| Masalah | Solusi |
|---|---|
| `~/.claude/mcp.json` tidak dibaca Claude Code | File yang benar: `~/.claude.json`, dikelola via `claude mcp add -s user` |
| MCP connect gagal — HTTPS vs HTTP | `OBSIDIAN_HOST` ke HTTPS (27124) tapi HTTP dimatikan → enable HTTP di Local REST API plugin |
| Config lama aktif setelah re-register | Butuh restart Claude Code agar `mcp-server.exe` jalan ulang dengan env baru |
| `OBSIDIAN_HOST` override ke HTTPS | Dihapus dari config — server default ke HTTP port 27123 |

### Verifikasi Koneksi
Setelah restart Claude Code, MCP berhasil:
- `list_vault_files` → root vault terdeteksi (`AI Software Factory/`, `Projects/`, dll)
- `get_vault_file` → berhasil baca `Projects/tanah-yang-dijanjikan.md`
- Vault path: `D:\Side-Mission\knowledge-vault\Claude-Knowledge`

---

## Related
- [[Development Log/2026-04-26]] — WA automation POC + MCP Obsidian setup session
