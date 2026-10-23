# CLAUDE.md
## Panduan AI untuk Codebase HRD System · ABPS (v2.2)

Dokumen ini dibaca oleh Claude (AI assistant) setiap kali bekerja di repository ini. Semua aturan di sini **wajib diikuti** — tidak ada pengecualian kecuali ada instruksi eksplisit dari developer yang disertai alasan teknis yang valid.

---

## Identitas Proyek

```
Nama          : HRD System — Absensi & Payroll (ABPS / Ayam Bakar Pucuk Singkong)
Tenant awal   : abps — Outlet Baros + Dapur Pusat
Prototipe     : https://digihrd.vercel.app/
Framework     : Next.js 15 (App Router, SSR)
Database      : Neon PostgreSQL (serverless, raw SQL)
Deploy        : Vercel
Storage       : Vercel Blob
Auth          : next-auth v5
UI            : shadcn/ui + lucide-react + Tailwind CSS v4
```

Baca `TRD.md` untuk detail teknis lengkap. Baca `ERD.md` untuk skema database. Baca `PRD.md` §8 untuk aturan payroll ABPS. Keputusan konfigurasi (Q/A → setting key → opsi → default) ada di `CHANGELOG-analisa.md` §5 dan `PRD.md` §17.3. Mekanisme rule engine ada di `TRD.md` §15.

### Glosarium Domain ABPS
| Istilah | Arti |
|---|---|
| Pay group | Kelompok gaji: `WEEKLY_PUSAT` (Sabtu–Jumat), `MONTHLY_OUTLET`, `MONTHLY_KANTOR`, `PART_TIME` |
| Gaji harian ÷ 9 | Upah per jam produksi pusat |
| Jam hitung | MAX(jam masuk, 05:00) untuk produksi pusat |
| Basis 26 | Gaji per hari karyawan bulanan = gaji bulan ÷ 26 |
| Tunj. Tidak Tetap | Tunjangan kehadiran & transport (biasanya 50% gaji) |
| Kasbon | Pinjaman/uang muka gaji, dipotong saat payroll |
| Borongan | Upah per kg (metik cengek/baput) atau per tugas (kurir) |
| Gaji gantungan | Gaji yang ditahan dan dilepas kemudian |
| Omset bersih | Omset kotor − diskon − potongan ojol (dasar bonus outlet) |
| Station | ck checker, kit/kc kitchen, ks kasir, se server |

---

## Aturan Absolute — Jangan Pernah Dilanggar

### 1. Tidak Ada ORM di Runtime

```typescript
// ❌ DILARANG KERAS — jangan pernah tulis ini
import { db } from '@/lib/db';
const employees = await db.select().from(schema.employees).where(...);

// ✅ WAJIB — selalu gunakan raw query
import { getEmployeeList } from '@/lib/db/queries/employees';
const employees = await getEmployeeList({ tenantId, search });
```

Drizzle (`drizzle-orm`) hanya boleh diimport di:
- `drizzle/schema.ts`
- `drizzle/seed.ts`
- `drizzle.config.ts`

**Tidak boleh diimport di tempat lain.**

---

### 2. Tidak Ada SQL di Luar `lib/db/queries/`

```
lib/db/queries/employees.ts     ← SQL boleh di sini
lib/db/queries/attendances.ts   ← SQL boleh di sini
lib/db/queries/payroll.ts       ← SQL boleh di sini

app/api/employees/route.ts      ← ❌ tidak boleh ada SQL di sini
app/admin/karyawan/page.tsx     ← ❌ tidak boleh ada SQL di sini
components/employee-table.tsx   ← ❌ tidak boleh ada SQL di sini
```

Jika diminta menambahkan query baru, buat function baru di file query yang sesuai, lalu import ke API route.

---

### 3. Tidak Ada `"use client"` Kecuali Perlu

Semua file default ke Server Component. Tambahkan `"use client"` hanya jika membutuhkan:
- `useState`, `useEffect`, `useRef`, `useCallback` (React hooks)
- Browser API (`navigator.geolocation`, `window`, `document`)
- Event handler yang butuh state (`onClick` dengan state update)
- Library yang perlu DOM: `recharts`, `@dnd-kit`, `@tiptap/react`

```typescript
// ❌ Jangan tambahkan "use client" hanya karena tidak yakin
'use client'; // tidak perlu!
export function EmployeeTable({ employees }) {
  return <table>...</table>; // tidak ada hooks, bisa jadi Server Component
}

// ✅ Server Component cukup
export function EmployeeTable({ employees }) {
  return <table>...</table>;
}
```

---

### 4. Tidak Ada Data Fetching di Client Components

```typescript
// ❌ DILARANG — fetch di useEffect
'use client';
export function EmployeeList() {
  const [employees, setEmployees] = useState([]);
  useEffect(() => {
    fetch('/api/employees').then(r => r.json()).then(setEmployees);
  }, []);
}

// ✅ WAJIB — fetch di Server Component, pass sebagai props
// app/admin/karyawan/page.tsx (Server Component)
export default async function KaryawanPage() {
  const employees = await getEmployeeList({ tenantId });
  return <EmployeeTable employees={employees} />;
}
```

Pengecualian yang boleh: mutasi (POST/PATCH/DELETE) via `fetch` atau Server Actions di dalam `"use client"` component, untuk aksi user yang tidak perlu SSR.

---

### 5. Selalu Validasi Input di API Route dengan Zod

```typescript
// ❌ Tidak ada validasi
const body = await req.json();
await createEmployee(body); // berbahaya

// ✅ Selalu parse dengan zod dulu
const parsed = createEmployeeSchema.safeParse(body);
if (!parsed.success) {
  return NextResponse.json({ error: parsed.error.flatten() }, { status: 422 });
}
await createEmployee(parsed.data);
```

---

### 6. Selalu Cek Session & Tenant di Setiap API Route

Setiap API route yang mengakses database harus:
1. Cek session (`getServerSession`)
2. Extract `tenantId` dari session, **bukan dari request body/params**
3. Tambahkan `tenant_id = ${tenantId}` ke semua query — tidak boleh ada query lintas tenant

```typescript
// ❌ tenantId dari request — bisa dimanipulasi
const { tenantId } = await req.json(); // BAHAYA

// ✅ tenantId selalu dari session
const session = await getServerSession(authConfig);
const tenantId = BigInt(session.user.tenantId); // AMAN
```

---

### 7. Semua File Upload ke Vercel Blob

```typescript
// ❌ Jangan simpan file di filesystem atau public/
fs.writeFileSync('./public/uploads/' + filename, buffer);

// ✅ Selalu upload ke Vercel Blob
import { uploadFile } from '@/lib/blob/upload';
const url = await uploadFile(file, 'attendance-photos', filename);
```

---

### 8. Tidak Ada Angka, Toggle, atau Pilihan Metode Aturan di Kode — Semua Lewat Resolver
Angka seperti 26, 9, 05:00, 500/menit, 10.000/jam, 17.000/jam, 30 menit grace, dan tier 350/400 jt, **dan juga** toggle (BPJS, PPh21, gantungan, catering, insentif, Lebaran 50%) serta pilihan metode (alokasi bonus, rounding, total-hours basis, sumber omset), **wajib** diambil dari `resolveRules()` (`lib/rules/resolver.ts`) atau dari snapshot run. Hardcode hanya boleh di seed (`drizzle/seed/settings.ts`) dan test fixture. Calculator tidak boleh punya nilai default sendiri. Query ke `setting_values` hanya boleh di `lib/db/queries/settings.ts`.

```typescript
// ❌ SALAH
if (emp.payGroup === 'WEEKLY_PUSAT' && clockIn < '05:00') bonus += 10000;
// ✅ BENAR
const r = snapshot.rulesFor(emp.id);
if (r['feature.early_arrival_incentive'].v && clockIn < r['weekly.earliest_count_time'].v)
  bonus += r['weekly.early_arrival_amount'].v;
```
Opsi baru = `setting_options` + strategi di `lib/rules/strategies/` + test. Jangan tambah `if/else` di calculator.

```typescript
// ❌ SALAH
const perDay = salary / 26;
// ✅ BENAR
const perDay = salary / rules.divisorDaysPerMonth;
```

### 9. Data Pribadi Tidak Bocor
- Jangan kembalikan `nik`, `npwp`, `bank_account_number`, `phone`, `password_hash` di response list/umum.
- Jangan log data pribadi; jangan masukkan data asli ke seed, fixture, atau prompt AI.
- Prototipe saat ini membuka `/api/*` tanpa login — setiap route baru/yang disentuh **wajib** `requireSession()` + filter `tenant_id`.

### 10. Payroll Run yang Sudah Approved Tidak Boleh Diubah
Koreksi dibuat sebagai run/adjustment baru. Semua perubahan nominal manual dicatat di `audit_logs` beserta alasan.

### 11. BPJS & PPh21 di Balik Feature Flag
Setting `feature.bpjs` / `feature.pph21` (default **off** untuk ABPS). Jangan menambah potongan BPJS/PPh21 tanpa flag aktif hasil resolver.

### 12. Setiap Perubahan Setting = Versi Baru + Audit
Jangan pernah `UPDATE` nilai setting yang sudah aktif. Buat versi baru (`effective_from`), tutup versi lama, tulis `setting_audit_logs` dalam satu transaksi, dan wajib isi alasan. Tolak perubahan yang masuk ke periode run approved/paid (gunakan run adjustment).

### 12a. Persetujuan Selalu Dinamis (v2.2)
- Perlu persetujuan atau tidak, siapa penyetujunya, dan berapa jumlahnya **hanya** ditentukan oleh `resolveApprovalPolicy()` (`lib/rules/approval.ts`). Jangan membaca kolom `requires_approval` atau hardcode `role === 'owner'` di route/komponen.
- Mengubah `approval_policies` selalu membuat approval request meta (Owner + HR Admin). Tidak ada endpoint/flag/env untuk mem-bypass.
- Tidak ada self-approval. Key ber-floor dilonggarkan hanya dengan 2 penyetuju. Semuanya ke `setting_audit_logs`.
- `system.public_api` dan `meta.approval_policy` terkunci: jangan buat UI/API untuk mengubahnya.

### 13. Payroll Hanya Membaca Snapshot
Generate run → `buildSnapshot()`. Kalkulasi, slip, dan laporan run membaca `payroll_run_snapshots`, bukan setting live.

### 14. Jangan Memutuskan Aturan Bisnis Sendiri
Jika aturan ambigu, buat setting dengan opsi-opsinya, default = praktik sheet, lalu catat di `CHANGELOG-analisa.md` §5.

---

## Panduan per Modul

### Ketika diminta menambah fitur baru:

1. **Tentukan apakah perlu tabel baru** → jika ya, tambahkan di `drizzle/schema.ts` dan buat migration
2. **Buat query functions** di `lib/db/queries/{model}.ts`
3. **Buat/update API route** di `app/api/{model}/route.ts`
4. **Buat Server Component** untuk page di `app/(dashboard)/...`
5. **Buat Client Component** hanya untuk bagian interaktif
6. **Tambahkan ke sidebar nav** jika fitur baru butuh menu

### Ketika diminta membuat query baru:

- Selalu gunakan parameterized query dengan template literal (bukan string concatenation)
- Selalu sertakan `tenant_id` filter
- Selalu sertakan `deleted_at IS NULL` untuk tabel dengan soft delete
- Gunakan `BIGINT` untuk semua ID, bukan `number` di TypeScript
- Kembalikan tipe yang spesifik, bukan `any[]`

```typescript
// Template query yang benar:
export async function getSomething(params: { tenantId: bigint; id: bigint }): Promise<SomeType | null> {
  const sql = getDb();
  const rows = await sql<SomeType[]>`
    SELECT *
    FROM some_table
    WHERE id = ${params.id}
      AND tenant_id = ${params.tenantId}
      AND deleted_at IS NULL
    LIMIT 1
  `;
  return rows[0] ?? null;
}
```

### Ketika diminta membuat komponen UI:

- Selalu gunakan komponen dari `components/ui/` (shadcn/ui) sebagai base
- Gunakan `lucide-react` untuk semua ikon
- Styling hanya dengan Tailwind utility classes
- Gunakan `cn()` dari `lib/utils` untuk conditional classes
- Ikuti naming convention: `kebab-case.tsx` untuk file, `PascalCase` untuk component name

```typescript
// Contoh komponen yang benar:
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Users } from 'lucide-react';
import { cn } from '@/lib/utils';

interface StatusBadgeProps {
  status: 'present' | 'late' | 'absent';
  className?: string;
}

const STATUS_MAP = {
  present: { label: 'Hadir',     variant: 'success' as const },
  late:    { label: 'Terlambat', variant: 'warning' as const },
  absent:  { label: 'Absen',     variant: 'destructive' as const },
};

export function StatusBadge({ status, className }: StatusBadgeProps) {
  const { label, variant } = STATUS_MAP[status];
  return (
    <Badge variant={variant} className={cn('text-xs', className)}>
      {label}
    </Badge>
  );
}
```

---

## File yang Tidak Boleh Diedit Langsung

| File | Alasan |
|---|---|
| `drizzle/migrations/*` | Auto-generated oleh drizzle-kit. Edit via `schema.ts` |
| `components/ui/*` | Auto-generated oleh shadcn/ui CLI. Update via `npx shadcn add` |
| `.next/*` | Build artifacts |
| `node_modules/*` | Dependencies |

---

## Cara Menambah shadcn Component Baru

```bash
# Jangan buat manual, selalu gunakan CLI
npx shadcn@latest add [component-name]

# Contoh:
npx shadcn@latest add calendar
npx shadcn@latest add data-table
npx shadcn@latest add date-picker
```

---

## Cara Menambah Table/Column Baru

```bash
# 1. Edit drizzle/schema.ts
# 2. Generate migration
npx drizzle-kit generate

# 3. Review SQL yang di-generate di drizzle/migrations/
# 4. Apply ke database
npx drizzle-kit migrate

# 5. Update types di lib/utils/types.ts jika perlu
# 6. Buat/update query functions di lib/db/queries/
```

---

## Cara Debug Query

Jika ada query yang lambat atau error:

```typescript
// Tambahkan console.log sementara untuk debug
export async function getProblemQuery(params) {
  const sql = getDb();
  console.time('getProblemQuery');  // debug timing

  const result = await sql`...`;

  console.timeEnd('getProblemQuery');
  console.log('rows:', result.length);
  return result;
}
```

Untuk EXPLAIN ANALYZE di Neon console:
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... FROM attendances WHERE ...;
```

---

## Respons Default untuk Pertanyaan Umum

**"Apakah saya bisa pakai Prisma/Drizzle ORM untuk query runtime?"**
Tidak. Gunakan raw SQL di `lib/db/queries/`. Lihat contoh di TRD.md Section 4.

**"Boleh saya tambah `"use client"` di page.tsx?"**
Tidak boleh. Page harus Server Component. Buat sub-komponen terpisah yang `"use client"` jika ada bagian yang butuh interaktivitas.

**"Bagaimana cara fetch data di Client Component?"**
Data di-fetch di Server Component (page.tsx), lalu di-pass sebagai props ke Client Component. Client Component hanya boleh fetch untuk mutasi (POST/PATCH/DELETE).

**"Boleh saya simpan file di `public/uploads/`?"**
Tidak. Semua file harus ke Vercel Blob via `lib/blob/upload.ts`.

**"Apakah boleh buat custom CSS class?"**
Hanya jika Tailwind benar-benar tidak bisa menanganinya. Gunakan `@layer utilities` di `globals.css`, bukan inline `<style>` atau CSS modules.
