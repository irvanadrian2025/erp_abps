# AGENTS.md
## Panduan Pembagian Tugas — AI Agents & Tim Developer · ABPS HRD (v2.2)

> v2.0 (24 Okt 2026): diselaraskan dengan data lapangan Ayam Bakar Pucuk Singkong dan prototipe https://digihrd.vercel.app/. Aturan bisnis payroll = `PRD.md` §8; skema = `ERD.md`; calculator = `TRD.md` §8.3.

Dokumen ini mendefinisikan bagaimana pekerjaan dibagi, baik antara developer manusia maupun AI agents (Claude, Cursor, Copilot, atau tools sejenis), agar tidak terjadi konflik, duplikasi, atau pelanggaran arsitektur.

---

## Prinsip Dasar

Setiap agent atau developer harus:

1. **Baca `CLAUDE.md` sebelum menulis kode apapun** — berisi aturan arsitektur yang tidak boleh dilanggar
2. **Baca `TRD.md` sebelum mengerjakan fitur baru** — berisi pola teknis yang wajib diikuti
3. **Baca `ERD.md` sebelum menyentuh database** — berisi skema dan relasi tabel
4. **Tidak berasumsi tentang struktur yang ada** — selalu cek file yang sudah ada sebelum membuat yang baru
5. **Tidak mengarang aturan payroll** — jika aturan tidak ada di PRD §8 / `pay_rules`, berhenti dan catat sebagai pertanyaan (format `Q-xx`) di `CHANGELOG-analisa.md`
6. **Uji calculator dengan fixture sheet asli** (TRD §8.3.6) sebelum PR payroll di-merge
7. **(v2.1) Semua aturan lewat resolver.** Dilarang hardcode angka, toggle, atau pilihan metode. Butuh aturan baru? Tambah `setting_definitions` (+ `setting_options`, strategi, test), lalu baca via `resolveRules()` / snapshot run (TRD §15)
8. **(v2.2) Persetujuan lewat `resolveApprovalPolicy()`.** Jangan membaca `setting_definitions.requires_approval` langsung, dan jangan hardcode role penyetuju. Perubahan kebijakan persetujuan selalu lewat meta-approval (TRD §16)
9. **(v2.1) Jangan memilih jawaban bisnis.** Ketidakjelasan diubah menjadi opsi + default dari sheet, bukan keputusan sepihak
10. **Tidak memasukkan data pribadi asli** (NIK, KK, rekening, HP) ke repo, seed, fixture, log, atau prompt AI

---

## Pembagian Domain Pekerjaan

### Domain A — Database & Query Layer
**File yang terlibat:** `drizzle/schema.ts`, `drizzle/migrations/*`, `drizzle/seed.ts`, `lib/db/queries/*`

**Siapa yang mengerjakan:** Developer backend atau AI agent yang diberi instruksi eksplisit untuk mengerjakan query

**Aturan:**
- Satu developer/agent hanya mengerjakan satu file query pada satu waktu untuk menghindari konflik
- Setiap fungsi query harus di-export dan diberi JSDoc minimal satu baris
- Setelah menambah query baru, update juga tipe di `lib/utils/types.ts`
- Migration hanya di-generate setelah `drizzle/schema.ts` final — jangan generate migration dari schema yang masih WIP

**Checklist sebelum commit:**
```
□ Query menggunakan template literal (bukan string concat)
□ Semua query include filter tenant_id
□ Tabel dengan soft delete include WHERE deleted_at IS NULL
□ Return type di-annotate dengan TypeScript generic: sql<MyType[]>
□ Tidak ada query di luar lib/db/queries/
```

---

### Domain B — API Routes
**File yang terlibat:** `app/api/**/*`

**Aturan:**
- Satu API route file = satu resource (employees, attendances, payroll, dst)
- Setiap handler (`GET`, `POST`, `PATCH`, `DELETE`) harus:
  1. Cek session
  2. Cek role
  3. Parse + validasi input dengan Zod
  4. Panggil query function dari `lib/db/queries/`
  5. Handle error dengan try/catch
  6. Return respons JSON yang konsisten

**Format respons yang konsisten:**
```typescript
// Success
{ data: T }                        // GET single
{ data: T[] }                      // GET list
{ data: T, message: string }       // POST (201)
{ message: string }                // PATCH, DELETE (200)

// Error
{ error: string }                  // 4xx, 5xx
{ error: ZodFlattenedError }       // 422 validation
```

**Checklist sebelum commit:**
```
□ Session dicek di awal handler
□ Role dicek dengan requireRole()
□ Input divalidasi dengan Zod sebelum ke DB
□ tenantId diambil dari session, bukan dari request
□ Error di-catch dan di-log dengan konteks yang cukup
□ Tidak ada logika bisnis di API route (pindahkan ke lib/)
```

---

### Domain C — Business Logic
**File yang terlibat:** `lib/payroll/*`, `lib/geo/*`, `lib/pdf/*`, `lib/excel/*`

**Aturan:**
- Semua fungsi di `lib/` harus **pure** atau hampir pure — tidak ada side effects langsung ke database
- `lib/payroll/calculator.ts` tidak boleh import dari `lib/db/` — hanya menerima data sebagai parameter
- Fungsi kalkulasi harus di-unit test

**Contoh pola yang benar:**
```typescript
// lib/payroll/calculator.ts — PURE, mudah di-test
export function calculatePayroll(input: PayrollInput): PayrollResult {
  // pure computation, tidak ada DB call
}

// app/api/payroll/runs/[runId]/calculate/route.ts — mengambil data, kalkulasi, simpan
const employees = await getEmployeesForPayroll(runId, tenantId);   // DB call
const results = employees.map(e => calculatePayroll(buildInput(e))); // pure logic
await savePayrollResults(runId, results);                           // DB call
```

---

### Domain D — Pages & Layouts
**File yang terlibat:** `app/(dashboard)/**/*`, `app/(auth)/**/*`

**Aturan:**
- Semua `page.tsx` dan `layout.tsx` adalah Server Component by default
- Page boleh memanggil query langsung (`import { getX } from '@/lib/db/queries/x'`) — ini adalah pola yang diizinkan untuk Server Component
- Logika rendering kompleks di-extract ke komponen terpisah di subfolder `components/`
- Setiap page harus export `metadata` untuk SEO

```typescript
// Wajib ada di setiap page
export const metadata: Metadata = {
  title: 'Manajemen Karyawan — HRD System',
  description: '...',
};
```

**Checklist sebelum commit:**
```
□ Page.tsx adalah Server Component (tidak ada "use client" di page level)
□ Data di-fetch di server (bukan useEffect)
□ Loading state menggunakan loading.tsx atau Suspense
□ Error state menggunakan error.tsx
□ Metadata di-export
□ Redirect jika tidak ada session
```

---

### Domain E — Shared Components
**File yang terlibat:** `components/shared/*`, `components/charts/*`

**Aturan:**
- Komponen di `components/shared/` harus **reusable** — tidak boleh ada hardcoded data atau business logic
- Komponen di `components/ui/` adalah milik shadcn — **tidak boleh diedit manual**
- Setiap komponen baru di `components/shared/` harus bisa bekerja dengan data apapun (generic props)
- Dokumentasikan props dengan JSDoc atau TypeScript interface yang jelas

**Template komponen shared:**
```typescript
// components/shared/stat-card.tsx
import { LucideIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

interface StatCardProps {
  label: string;
  value: string | number;
  icon: LucideIcon;
  description?: string;
  trend?: { value: number; direction: 'up' | 'down' };
  className?: string;
}

export function StatCard({ label, value, icon: Icon, description, trend, className }: StatCardProps) {
  // implementasi
}
```

---

## Pola Kerja AI Agent

### Ketika agent diminta mengerjakan task, langkah wajibnya:

```
1. BACA konteks → CLAUDE.md, TRD.md, ERD.md
2. IDENTIFIKASI file yang sudah ada yang relevan
3. TENTUKAN apa yang perlu dibuat/diubah
4. KERJAKAN dari layer paling dalam ke luar:
   a. Query function (lib/db/queries/)
   b. Types (lib/utils/types.ts)
   c. API route (app/api/)
   d. Server Component / page
   e. Client Component (jika perlu)
5. VERIFIKASI tidak ada aturan yang dilanggar
6. JELASKAN apa yang dibuat dan mengapa
```

### Cara mendeskripsikan task ke AI agent:

**Format yang direkomendasikan:**
```
Task: [nama fitur/perubahan]
File yang perlu dibuat/diedit: [list file]
Referensi model DB: [nama tabel dari ERD]
Catatan khusus: [aturan tambahan jika ada]

Contoh:
Task: Tambah endpoint untuk approve leave request
File: lib/db/queries/leaves.ts, app/api/leaves/[id]/approve/route.ts
Referensi DB: leave_requests (ERD.md)
Catatan: Hanya hr_admin dan hr_staff yang boleh approve
```

---

## Registri Tugas per Modul

Gunakan ini sebagai referensi siapa yang mengerjakan apa. Update setiap kali ada assignment baru.

### Modul: Auth & Tenant
| Task | File | Status | Owner |
|---|---|---|---|
| Login page (prototipe: 404 — PRIORITAS) | `app/(auth)/login/page.tsx` | - | - |
| next-auth config | `lib/auth/config.ts` | - | - |
| Middleware guard (prototipe: API terbuka publik — PRIORITAS) | `middleware.ts` | - | - |
| Tenant branding | `app/api/tenants/[id]/route.ts` | - | - |

### Modul: Karyawan
| Task | File | Status | Owner |
|---|---|---|---|
| Employee queries | `lib/db/queries/employees.ts` | - | - |
| Employee API | `app/api/employees/route.ts` | - | - |
| Employee list page | `app/admin/karyawan/page.tsx` | - | - |
| Employee form | `app/admin/karyawan/components/employee-form.tsx` | - | - |

### Modul: Absensi
| Task | File | Status | Owner |
|---|---|---|---|
| Attendance queries | `lib/db/queries/attendances.ts` | - | - |
| Clock-in API | `app/api/attendances/route.ts` | - | - |
| Geo helper | `lib/geo/distance.ts` | - | - |
| Clock-in widget | `app/employee/absensi/components/clock-in-widget.tsx` | - | - |
| Admin attendance table | `app/admin/absensi/page.tsx` | - | - |

### Modul: Shift
| Task | File | Status | Owner |
|---|---|---|---|
| Shift queries | `lib/db/queries/shifts.ts` | - | - |
| Shift API | `app/api/shifts/route.ts` | - | - |
| Shift scheduler (grid bulanan + station ck/kit/ks/se + off/izin/sakit) | `app/(admin)/shifts/schedule/components/shift-scheduler.tsx` | - | - |
| Swap request flow (wajib pengganti + konfirmasi Leader) | `app/api/shifts/swaps/route.ts` | - | - |

### Modul: Payroll
| Task | File | Status | Owner |
|---|---|---|---|
| Payroll queries | `lib/db/queries/payroll.ts` | - | - |
| Pay groups & pay rules queries | `lib/db/queries/pay-rules.ts` | - | - |
| Period helper (Sabtu–Jumat / bulanan) | `lib/payroll/period.ts` | - | - |
| Calculator mingguan pusat | `lib/payroll/weekly-hourly.ts` | - | - |
| Calculator bulanan (split 50/50, 50/30/20, …) | `lib/payroll/monthly-split.ts` | - | - |
| Freelance & part-time | `lib/payroll/freelance.ts`, `lib/payroll/part-time.ts` | - | - |
| Lembur (9 jam + 30 mnt, roundup, flat) | `lib/payroll/overtime.ts` | - | - |
| Potongan (telat/izin/sakit/prorate/kasbon/gantungan) | `lib/payroll/deductions.ts` | - | - |
| Bonus omset | `lib/payroll/bonus.ts` | - | - |
| PPh21 / BPJS (feature flag, non-aktif ABPS) | `lib/payroll/pph21.ts`, `lib/payroll/bpjs.ts` | - | - |
| Fixture dari sheet asli | `lib/payroll/__tests__/fixtures/*.json` | - | - |
| Payroll run API | `app/api/payroll-runs/route.ts` (selaras prototipe) | - | - |
| Generate API | `app/api/payroll-runs/[id]/generate/route.ts` | - | - |
| Review/Approve/Paid API | `app/api/payroll-runs/[id]/(review|approve|mark-paid)/route.ts` | - | - |
| Payroll page (admin) | `app/(admin)/payroll/page.tsx` | - | - |

### Modul: Rule Engine & Pengaturan Dinamis (baru v2.1)
| Task | File | Status | Owner |
|---|---|---|---|
| Migration registry + seed definisi/opsi/default | `drizzle/schema.ts`, `drizzle/seed/settings.ts` | - | - |
| Settings queries (satu-satunya akses) | `lib/db/queries/settings.ts` | - | - |
| Resolver + dependency | `lib/rules/resolver.ts` | - | - |
| Validasi (zod + lintas setting + larangan retroaktif) | `lib/rules/validate.ts` | - | - |
| Strategi per opsi | `lib/rules/strategies/*.ts` | - | - |
| Snapshot run & trigger immutability | `lib/rules/snapshot.ts`, migration trigger | - | - |
| Typegen key | `scripts/rules-gen.ts` (`pnpm rules:gen`) | - | - |
| API settings (definitions/resolved/values/approve/preview/audit) | `app/api/settings/**` | - | - |
| Layar Pengaturan Aturan + override + riwayat + preview | `app/(admin)/settings/rules/**` | - | - |
| Penjelasan nilai per karyawan | `app/(admin)/employees/[id]/rules/page.tsx` | - | - |
| Audit log viewer | `app/(admin)/settings/audit/page.tsx` | - | - |
| (v2.2) Approval policy resolver & alur decide | `lib/rules/approval.ts`, `lib/db/queries/approvals.ts` | - | - |
| (v2.2) API approval policies & approvals | `app/api/settings/approval-policies/**`, `app/api/approvals/**` | - | - |
| (v2.2) Layar Kebijakan Persetujuan & Kotak Persetujuan | `app/(admin)/settings/approval-policies/**`, `app/(admin)/approvals/**` | - | - |
| (v2.2) Seed default policies + guardrail (trigger/constraint) | `drizzle/seed/approval-policies.ts`, migration | - | - |
| Import omset (MOKA/Runchise/Kledo/ojol) | `app/api/revenue-imports/route.ts`, `lib/import/revenue/*.ts` | - | - |

### Modul: Kasbon (baru v2.0)
| Task | File | Status | Owner |
|---|---|---|---|
| Queries | `lib/db/queries/cash-advances.ts` | - | - |
| API | `app/api/cash-advances/route.ts` | - | - |
| Page | `app/(admin)/kasbon/page.tsx` | - | - |

### Modul: Borongan / Kurir / Kunjungan (baru v2.0)
| Task | File | Status | Owner |
|---|---|---|---|
| Queries | `lib/db/queries/piece-work.ts`, `lib/db/queries/visits.ts` | - | - |
| API | `app/api/piece-work-entries/route.ts`, `app/api/visit-logs/route.ts` | - | - |
| Input harian (Leader/Admin) | `app/(admin)/borongan/page.tsx` | - | - |

### Modul: Omset & Bonus (baru v2.0)
| Task | File | Status | Owner |
|---|---|---|---|
| Queries | `lib/db/queries/revenues.ts`, `lib/db/queries/bonuses.ts` | - | - |
| API | `app/api/outlet-revenues/route.ts`, `app/api/bonus-pools/route.ts` | - | - |
| Page tier & alokasi | `app/(admin)/bonus/page.tsx` | - | - |

### Modul: Pengaturan Payroll (baru v2.0)
| Task | File | Status | Owner |
|---|---|---|---|
| Pay groups & pay rules (berversi) | `app/(admin)/settings/pay-rules/page.tsx` | - | - |
| Tanggal merah & libur operasional | `app/(admin)/settings/holidays/page.tsx` | - | - |
| Tarif borongan | `app/(admin)/settings/piece-rates/page.tsx` | - | - |
| Import Excel karyawan & gaji | `app/api/import/employees/route.ts` | - | - |

### Modul: Slip Gaji
| Task | File | Status | Owner |
|---|---|---|---|
| PDF template bulanan (format SLIP GAJI ABPS) | `lib/pdf/slip-monthly.tsx` | - | - |
| PDF template mingguan (harian Sabtu–Jumat) | `lib/pdf/slip-weekly.tsx` | - | - |
| Slip generate API | `app/api/payroll/items/[itemId]/slip/route.ts` | - | - |
| Slip list (admin) | `app/admin/slip-gaji/page.tsx` | - | - |
| Slip view (employee) | `app/employee/slip-gaji/page.tsx` | - | - |

### Modul: Laporan
| Task | File | Status | Owner |
|---|---|---|---|
| Excel generators | `lib/excel/generators.ts` | - | - |
| Attendance report API | `app/api/reports/attendance/route.ts` | - | - |
| Payroll report API | `app/api/reports/payroll/route.ts` | - | - |
| Report page | `app/admin/laporan/page.tsx` | - | - |

### Modul: Izin & Cuti
| Task | File | Status | Owner |
|---|---|---|---|
| Leave queries | `lib/db/queries/leaves.ts` | - | - |
| Leave API | `app/api/leaves/route.ts` | - | - |
| Leave form (tiptap) | `app/employee/izin/components/leave-form.tsx` | - | - |
| Leave approval | `app/api/leaves/[id]/approve/route.ts` | - | - |

### Modul: Upload & Storage
| Task | File | Status | Owner |
|---|---|---|---|
| Blob helper | `lib/blob/upload.ts` | - | - |
| Upload API | `app/api/upload/route.ts` | - | - |

### Modul: Dashboard & Charts
| Task | File | Status | Owner |
|---|---|---|---|
| Attendance bar chart | `components/charts/attendance-bar.tsx` | - | - |
| Payroll trend chart | `components/charts/payroll-trend.tsx` | - | - |
| Admin dashboard | `app/admin/page.tsx` | - | - |
| Employee dashboard | `app/employee/page.tsx` | - | - |

---

## Prioritas Pengerjaan v2.0

1. **Keamanan prototipe**: login + middleware + tenant/role check di semua `/api/*`; hapus data dummy.
2. **Rule engine (resolver, registry, snapshot, audit)**: harus selesai sebelum calculator.
3. Master data ABPS: lokasi, pay groups, setting default, shift template, komponen gaji, import karyawan.
4. Presensi + jadwal (grid bulanan, station, off).
5. Calculator mingguan pusat + fixture → payroll run mingguan + slip mingguan.
6. Calculator bulanan + kasbon + izin/sakit + lembur → run bulanan + slip bulanan.
7. Borongan, kunjungan, bonus omset, gaji gantungan.
8. Laporan Excel format sheet lama; paralel run 1 bulan.

---

## Konflik Resolution

Jika dua agent atau developer mengerjakan file yang saling berkaitan:

1. **Definisikan interface/type dulu** sebelum implementasi — sepakati bentuk data yang dipertukarkan
2. **API route adalah kontrak** — jika API route sudah define response shape, komponen yang consume tidak boleh mengubahnya sendiri
3. **Query functions adalah single source of truth** — jika ada bug di data, fix di query layer, bukan di komponen

---

## Checklist Pre-Commit Global

Sebelum commit apapun, pastikan:

```
□ npm run type-check  — tidak ada TypeScript error
□ npm run lint        — tidak ada ESLint warning/error
□ Tidak ada file dengan import 'drizzle-orm' di luar drizzle/ folder
□ Tidak ada raw SQL string di luar lib/db/queries/
□ Tidak ada "use client" di page.tsx atau layout.tsx
□ Tidak ada hardcoded tenantId atau userId
□ Tidak ada console.log yang tertinggal (kecuali di development-only code)
□ Semua file baru mengikuti naming convention (kebab-case untuk file, PascalCase untuk component)
```

---

## Pesan untuk AI Agents

Jika kamu adalah AI agent yang membaca dokumen ini:

- **Jangan berasumsi** bahwa sebuah pola yang umum di Next.js boleh dipakai jika bertentangan dengan aturan di atas
- **Jangan inisiasi ORM call** — meski Drizzle terpasang, gunakan hanya untuk migrasi
- **Tanya terlebih dahulu** jika ada keputusan arsitektur yang tidak tercakup di dokumen ini, daripada membuat keputusan sendiri
- **Satu task, satu fokus** — jangan refactor hal lain yang tidak diminta saat mengerjakan sebuah fitur
- **Jelaskan keputusan** — jika membuat pilihan teknis (misal memilih antara dua pendekatan), jelaskan alasannya
- **Berikan kode yang lengkap** — jangan menghilangkan bagian dengan komentar seperti `// ... rest of implementation` jika diminta mengerjakan suatu file. Tulis semuanya.
