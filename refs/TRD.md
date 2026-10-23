# Technical Requirements Document (TRD)
## Sistem HRD — Absensi & Payroll · Ayam Bakar Pucuk Singkong (ABPS)
**Versi:** 2.2
**Status:** Draft revisi (data lapangan + audit prototipe digihrd.vercel.app)
**Tanggal:** 24 Oktober 2026

> **v2.2:** Approval policy engine dinamis (§16): resolver kebijakan persetujuan, multi-penyetuju, meta-approval, floor, dan audit. §15.5 langkah 6 kini memanggil §16.
>
> **v2.1:** Rule engine & resolver pengaturan dinamis + snapshot payroll (§15). Calculator §8.3 kini menerima `ResolvedRules` dari resolver/snapshot; contoh angka di §8.3 = default.
>
> **Perubahan utama v2.0:** payroll mingguan & bulanan (`pay_groups`), calculator ABPS (§8.3), kasbon/borongan/bonus (§8.3.7), lifecycle run per periode (§8.3.8), BPJS/PPh21 di balik feature flag, dan **temuan keamanan prototipe** (§6.4). Detail di `CHANGELOG-analisa.md`.

---

## 1. Prinsip Arsitektur

Dokumen ini mendefinisikan standar teknis yang **wajib diikuti** oleh seluruh anggota tim tanpa pengecualian. Setiap keputusan teknis yang tidak tercakup di sini harus didiskusikan dan dicatat sebagai amendment sebelum diimplementasikan.

### 1.1 Aturan Utama (Non-Negotiable)

| # | Aturan | Keterangan |
|---|---|---|
| R-01 | **Tidak ada SPA** | Seluruh aplikasi dibangun dengan Next.js SSR. HTML dan JSX yang sudah dibuat hanya bersifat referensi desain, bukan kode produksi |
| R-02 | **Tidak ada ORM di query runtime** | Drizzle hanya digunakan untuk migrasi dan seed. Semua query runtime menggunakan raw SQL via Neon serverless driver |
| R-03 | **Tidak ada raw query di komponen FE** | Seluruh akses database hanya melalui `app/api/{model}/route.ts`. Komponen React tidak boleh menyentuh database secara langsung |
| R-04 | **Server Components by default** | Semua page dan layout adalah React Server Component kecuali ada interaktivitas eksplisit yang membutuhkan `"use client"` |
| R-05 | **Vercel sebagai satu-satunya platform deploy** | Tidak ada self-hosted, tidak ada Docker untuk produksi |
| R-06 | **Vercel Blob untuk semua file** | Foto absensi, slip gaji PDF, logo perusahaan, dan dokumen lain disimpan di Vercel Blob, bukan di folder `public/` |
| R-07 | **Neon Postgres serverless** | Tidak ada koneksi database persistent. Setiap API route menggunakan koneksi serverless yang di-pool |

---

## 2. Tech Stack

### 2.1 Core

```
Framework      : Next.js 15 (App Router, SSR)
Runtime        : Node.js 20 LTS (Vercel Edge Runtime untuk middleware)
Language       : TypeScript 5 (strict mode)
Database       : Neon Postgres (serverless, pooled connections)
Deploy         : Vercel (production + preview deployments)
Storage        : Vercel Blob Storage
```

### 2.2 Frontend

```
UI Framework   : React 19 (Server Components default)
Component Lib  : shadcn/ui (Radix UI primitives)
Icons          : lucide-react
Styling        : Tailwind CSS v4
Font           : Cabin (heading), Albert Sans (body) — via next/font/google
```

### 2.3 Library Wajib

```
PDF            : @react-pdf/renderer  — generate slip gaji, laporan
Excel          : xlsx (SheetJS)       — export rekap kehadiran, payroll
Drag & Drop    : @dnd-kit/core + @dnd-kit/sortable — jadwal shift, kanban
Charts         : recharts             — dashboard kehadiran, payroll trend
Rich Text      : @tiptap/react        — catatan HR, announcement, alasan izin
Migration      : drizzle-kit          — schema migration & seed ONLY
Database       : @neondatabase/serverless — raw query driver
Validation     : zod                  — schema validation di API routes
Auth           : next-auth v5 (Auth.js) — session, JWT, role guard
```

### 2.4 Development Tools

```
Linter         : ESLint + @typescript-eslint
Formatter      : Prettier
Git hooks      : husky + lint-staged
Testing        : Vitest (unit), Playwright (e2e critical paths)
CI/CD          : Vercel + GitHub Actions (lint, type-check, test)
```

---

## 3. Struktur Direktori

```
app/
├── (auth)/
│   ├── login/
│   │   └── page.tsx                    # SSR login page
│   └── layout.tsx
│
├── (dashboard)/
│   ├── layout.tsx                      # Shell: sidebar + topbar (Server Component)
│   │
│   ├── admin/
│   │   ├── page.tsx                    # Dashboard HR
│   │   ├── karyawan/
│   │   │   ├── page.tsx                # Tabel karyawan (SSR)
│   │   │   ├── [id]/page.tsx           # Detail karyawan
│   │   │   └── components/
│   │   │       ├── employee-table.tsx  # Server Component
│   │   │       ├── employee-form.tsx   # "use client" — form interaktif
│   │   │       └── employee-filter.tsx # "use client" — filter/search
│   │   ├── absensi/
│   │   │   ├── page.tsx
│   │   │   └── components/
│   │   ├── shift/
│   │   │   ├── page.tsx
│   │   │   └── components/
│   │   │       └── shift-scheduler.tsx # "use client" — dnd-kit
│   │   ├── payroll/
│   │   │   ├── page.tsx
│   │   │   ├── [runId]/page.tsx
│   │   │   └── components/
│   │   ├── slip-gaji/
│   │   │   ├── page.tsx
│   │   │   └── components/
│   │   │       └── slip-pdf.tsx        # @react-pdf/renderer
│   │   └── pengaturan/
│   │       └── page.tsx
│   │
│   └── employee/
│       ├── page.tsx                    # Beranda karyawan
│       ├── absensi/
│       │   ├── page.tsx
│       │   └── components/
│       │       └── clock-in-widget.tsx # "use client" — GPS + geolocation
│       ├── jadwal/page.tsx
│       ├── izin/
│       │   ├── page.tsx
│       │   └── components/
│       │       └── leave-form.tsx      # "use client" + tiptap
│       └── slip-gaji/page.tsx
│
├── api/
│   ├── auth/[...nextauth]/route.ts
│   │
│   ├── employees/
│   │   ├── route.ts                    # GET (list), POST (create)
│   │   └── [id]/
│   │       └── route.ts                # GET, PATCH, DELETE
│   │
│   ├── attendances/
│   │   ├── route.ts                    # GET (list), POST (clock-in)
│   │   ├── [id]/route.ts               # GET, PATCH (koreksi)
│   │   └── clock-out/route.ts          # POST clock-out
│   │
│   ├── shifts/
│   │   ├── route.ts
│   │   ├── [id]/route.ts
│   │   ├── assignments/route.ts
│   │   └── swaps/route.ts
│   │
│   ├── payroll/
│   │   ├── runs/
│   │   │   ├── route.ts                # GET list, POST generate run
│   │   │   └── [runId]/
│   │   │       ├── route.ts            # GET, PATCH status
│   │   │       ├── calculate/route.ts  # POST trigger kalkulasi
│   │   │       └── approve/route.ts    # POST approval
│   │   └── items/
│   │       ├── route.ts
│   │       └── [itemId]/route.ts
│   │
│   ├── leaves/
│   │   ├── route.ts
│   │   └── [id]/
│   │       ├── route.ts
│   │       └── approve/route.ts
│   │
│   ├── overtime/
│   │   ├── route.ts
│   │   └── [id]/approve/route.ts
│   │
│   ├── locations/
│   │   ├── route.ts
│   │   └── [id]/route.ts
│   │
│   ├── reports/
│   │   ├── attendance/route.ts         # GET → JSON / Excel export
│   │   └── payroll/route.ts
│   │
│   ├── upload/
│   │   └── route.ts                    # POST → Vercel Blob
│   │
│   └── notifications/
│       └── route.ts
│
├── lib/
│   ├── db/
│   │   ├── index.ts                    # Neon pool singleton
│   │   └── queries/                    # SEMUA raw SQL ada di sini
│   │       ├── employees.ts
│   │       ├── attendances.ts
│   │       ├── shifts.ts
│   │       ├── payroll.ts
│   │       ├── leaves.ts
│   │       ├── locations.ts
│   │       └── reports.ts
│   │
│   ├── auth/
│   │   ├── config.ts                   # next-auth config
│   │   └── guards.ts                   # role-based access helpers
│   │
│   ├── payroll/
│   │   ├── calculator.ts               # Pure functions kalkulasi gaji
│   │   ├── pph21.ts                    # Kalkulasi PPh 21 (feature flag, non-aktif ABPS)
│   │   ├── bpjs.ts                     # Kalkulasi BPJS (feature flag, non-aktif ABPS)
│   │   ├── weekly-hourly.ts / monthly-split.ts / freelance.ts / part-time.ts
│   │   └── overtime.ts / bonus.ts / deductions.ts / period.ts   # v2.0, lihat §8.3
│   │
│   ├── pdf/
│   │   └── slip-template.tsx           # @react-pdf/renderer template
│   │
│   ├── excel/
│   │   └── generators.ts               # xlsx sheet generators
│   │
│   ├── geo/
│   │   └── distance.ts                 # Haversine formula + mock GPS detection
│   │
│   ├── blob/
│   │   └── upload.ts                   # Vercel Blob helpers
│   │
│   └── utils/
│       ├── format.ts                   # Currency, date, number formatters
│       ├── constants.ts
│       └── types.ts                    # Shared TypeScript types
│
├── components/
│   ├── ui/                             # shadcn/ui components (auto-generated)
│   │   ├── button.tsx
│   │   ├── table.tsx
│   │   ├── dialog.tsx
│   │   ├── badge.tsx
│   │   └── ...
│   │
│   ├── shared/                         # Reusable business components
│   │   ├── avatar-initials.tsx
│   │   ├── status-badge.tsx
│   │   ├── data-table.tsx              # Generic SSR-friendly table
│   │   ├── page-header.tsx
│   │   ├── stat-card.tsx
│   │   ├── empty-state.tsx
│   │   ├── loading-skeleton.tsx
│   │   ├── confirm-dialog.tsx
│   │   ├── date-range-picker.tsx       # "use client"
│   │   └── rich-text-editor.tsx        # "use client" — tiptap wrapper
│   │
│   └── charts/                         # recharts wrappers
│       ├── attendance-bar.tsx          # "use client"
│       ├── payroll-trend.tsx           # "use client"
│       └── presence-donut.tsx          # "use client"
│
├── drizzle/
│   ├── schema.ts                       # Schema definition (migration only)
│   ├── migrations/                     # Auto-generated migration files
│   └── seed.ts                         # Idempotent seed script
│
├── middleware.ts                       # Auth guard + role routing (Edge Runtime)
├── next.config.ts
├── tailwind.config.ts
├── drizzle.config.ts
└── .env.local / .env.production
```

---

## 4. Konvensi Database & Query Layer

### 4.1 Koneksi Database

```typescript
// lib/db/index.ts
import { neon } from '@neondatabase/serverless';

// Satu factory function — tidak ada singleton global
// Vercel serverless functions sudah handle connection pooling
export function getDb() {
  return neon(process.env.DATABASE_URL!);
}
```

### 4.2 Pola Raw Query — Wajib Diikuti

Semua query ditulis sebagai named functions di file `lib/db/queries/{model}.ts`. Tidak boleh ada SQL string di luar direktori ini.

```typescript
// lib/db/queries/employees.ts

import { getDb } from '@/lib/db';
import type { Employee, EmployeeListParams } from '@/lib/utils/types';

export async function getEmployeeList(params: EmployeeListParams): Promise<Employee[]> {
  const sql = getDb();
  const { tenantId, search, departmentId, status, limit = 50, offset = 0 } = params;

  return sql<Employee[]>`
    SELECT
      e.id,
      e.employee_number,
      e.full_name,
      e.email,
      e.phone,
      e.role,
      e.employment_status,
      e.join_date,
      e.is_active,
      p.name           AS position_name,
      d.name           AS department_name,
      eg.name          AS grade_name,
      pos.payroll_type,
      l.name           AS location_name
    FROM employees e
    JOIN positions p   ON p.id = e.position_id
    JOIN departments d ON d.id = p.department_id
    JOIN employment_grades eg ON eg.id = p.employment_grade_id
    LEFT JOIN locations l ON l.id = e.location_id
    WHERE e.tenant_id = ${tenantId}
      AND e.deleted_at IS NULL
      ${search ? sql`AND e.full_name ILIKE ${'%' + search + '%'}` : sql``}
      ${departmentId ? sql`AND d.id = ${departmentId}` : sql``}
      ${status ? sql`AND e.employment_status = ${status}` : sql``}
    ORDER BY e.full_name ASC
    LIMIT ${limit} OFFSET ${offset}
  `;
}

export async function getEmployeeById(id: bigint, tenantId: bigint): Promise<Employee | null> {
  const sql = getDb();
  const rows = await sql<Employee[]>`
    SELECT
      e.*,
      p.name           AS position_name,
      p.payroll_type,
      d.name           AS department_name,
      eg.name          AS grade_name,
      l.name           AS location_name,
      l.radius_meter
    FROM employees e
    JOIN positions p   ON p.id = e.position_id
    JOIN departments d ON d.id = p.department_id
    JOIN employment_grades eg ON eg.id = p.employment_grade_id
    LEFT JOIN locations l ON l.id = e.location_id
    WHERE e.id = ${id}
      AND e.tenant_id = ${tenantId}
      AND e.deleted_at IS NULL
    LIMIT 1
  `;
  return rows[0] ?? null;
}

export async function createEmployee(data: CreateEmployeeInput): Promise<{ id: bigint }> {
  const sql = getDb();
  const rows = await sql<{ id: bigint }[]>`
    INSERT INTO employees (
      tenant_id, position_id, location_id, employee_number, full_name,
      email, phone, nik, join_date, employment_status, marital_status,
      dependent_count, bank_name, bank_account_number, bank_account_name,
      base_salary, hourly_rate, overtime_rate_multiplier, ptkp_status,
      role, password_hash
    ) VALUES (
      ${data.tenantId}, ${data.positionId}, ${data.locationId},
      ${data.employeeNumber}, ${data.fullName}, ${data.email},
      ${data.phone}, ${data.nik}, ${data.joinDate}, ${data.employmentStatus},
      ${data.maritalStatus}, ${data.dependentCount}, ${data.bankName},
      ${data.bankAccountNumber}, ${data.bankAccountName}, ${data.baseSalary},
      ${data.hourlyRate}, ${data.overtimeRateMultiplier}, ${data.ptkpStatus},
      ${data.role}, ${data.passwordHash}
    )
    RETURNING id
  `;
  return rows[0];
}

export async function softDeleteEmployee(id: bigint, tenantId: bigint): Promise<void> {
  const sql = getDb();
  await sql`
    UPDATE employees
    SET deleted_at = NOW(), updated_at = NOW()
    WHERE id = ${id} AND tenant_id = ${tenantId}
  `;
}
```

### 4.3 API Route Pattern

```typescript
// app/api/employees/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authConfig } from '@/lib/auth/config';
import { requireRole } from '@/lib/auth/guards';
import { getEmployeeList, createEmployee } from '@/lib/db/queries/employees';
import { createEmployeeSchema } from '@/lib/utils/types';
import { z } from 'zod';

export async function GET(req: NextRequest) {
  const session = await getServerSession(authConfig);
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  requireRole(session, ['hr_admin', 'hr_staff', 'manager', 'supervisor']);

  const { searchParams } = req.nextUrl;
  const params = {
    tenantId: BigInt(session.user.tenantId),
    search: searchParams.get('search') ?? undefined,
    departmentId: searchParams.get('departmentId')
      ? BigInt(searchParams.get('departmentId')!)
      : undefined,
    status: searchParams.get('status') ?? undefined,
    limit: Number(searchParams.get('limit') ?? 50),
    offset: Number(searchParams.get('offset') ?? 0),
  };

  try {
    const data = await getEmployeeList(params);
    return NextResponse.json({ data });
  } catch (err) {
    console.error('[GET /api/employees]', err);
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}

export async function POST(req: NextRequest) {
  const session = await getServerSession(authConfig);
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  requireRole(session, ['hr_admin', 'hr_staff']);

  const body = await req.json();
  const parsed = createEmployeeSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.flatten() }, { status: 422 });
  }

  try {
    const result = await createEmployee({
      ...parsed.data,
      tenantId: BigInt(session.user.tenantId),
    });
    return NextResponse.json({ data: result }, { status: 201 });
  } catch (err) {
    console.error('[POST /api/employees]', err);
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}
```

### 4.4 Drizzle — Migrasi & Seed Saja

```typescript
// drizzle/schema.ts — ONLY used by drizzle-kit, never imported in runtime code

import { pgTable, bigserial, varchar, boolean, timestamp, numeric, integer, smallint, date, time, text, inet } from 'drizzle-orm/pg-core';

export const tenants = pgTable('tenants', {
  id:              bigserial('id', { mode: 'bigint' }).primaryKey(),
  name:            varchar('name', { length: 255 }).notNull(),
  slug:            varchar('slug', { length: 100 }).notNull().unique(),
  primaryColor:    varchar('primary_color', { length: 7 }).notNull().default('#1D4ED8'),
  secondaryColor:  varchar('secondary_color', { length: 7 }).notNull().default('#7C3AED'),
  timezone:        varchar('timezone', { length: 50 }).notNull().default('Asia/Jakarta'),
  isActive:        boolean('is_active').notNull().default(true),
  createdAt:       timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:       timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  deletedAt:       timestamp('deleted_at', { withTimezone: true }),
});

// ... other tables (same as ERD schema)
```

```typescript
// drizzle/seed.ts — Idempotent: safe to run multiple times

import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

const sql = neon(process.env.DATABASE_URL!);
const db = drizzle(sql, { schema });

async function seed() {
  console.log('Seeding...');

  // Idempotent: upsert pattern
  await db.insert(schema.tenants)
    .values({
      name: 'PT Maju Bersama Industri',
      slug: 'pt-maju-bersama',
      primaryColor: '#1D4ED8',
    })
    .onConflictDoNothing();

  // ... rest of seed
  console.log('Seed complete.');
}

seed().catch(console.error);
```

```bash
# drizzle.config.ts commands — tidak pernah dijalankan di runtime
npx drizzle-kit generate   # generate migration SQL
npx drizzle-kit migrate    # run migrations against Neon
npx tsx drizzle/seed.ts    # run seed (idempotent)
```

---

## 5. Rendering Strategy

### 5.1 Kapan SSR, Kapan Client Component

| Komponen | Strategi | Alasan |
|---|---|---|
| Page layouts, shell | Server Component | Tidak butuh interaktivitas, data dari DB |
| Tabel data (karyawan, absensi, payroll) | Server Component + Server Action | SSR = lebih cepat, SEO-friendly |
| Form tambah/edit | `"use client"` | Input, validation state, feedback |
| Clock-in widget | `"use client"` | Browser Geolocation API |
| Shift scheduler (drag & drop) | `"use client"` | dnd-kit membutuhkan DOM |
| Charts / recharts | `"use client"` | Canvas/SVG rendering |
| Rich text editor (tiptap) | `"use client"` | ProseMirror DOM |
| Date range picker | `"use client"` | Popover state |
| PDF viewer / trigger | `"use client"` | Blob URL handling |
| Modal / Dialog | `"use client"` | State open/close |
| Topbar notifikasi | `"use client"` | Realtime polling |

### 5.2 Data Fetching Pattern

```typescript
// app/admin/karyawan/page.tsx — Server Component (default)
// Data di-fetch langsung di server, bukan via useEffect

import { getServerSession } from 'next-auth';
import { authConfig } from '@/lib/auth/config';
import { getEmployeeList } from '@/lib/db/queries/employees';
import { EmployeeTable } from './components/employee-table';
import { redirect } from 'next/navigation';

interface PageProps {
  searchParams: { search?: string; dept?: string; page?: string };
}

export default async function KaryawanPage({ searchParams }: PageProps) {
  const session = await getServerSession(authConfig);
  if (!session) redirect('/login');

  const page = Number(searchParams.page ?? 1);
  const limit = 50;

  const employees = await getEmployeeList({
    tenantId: BigInt(session.user.tenantId),
    search: searchParams.search,
    departmentId: searchParams.dept ? BigInt(searchParams.dept) : undefined,
    limit,
    offset: (page - 1) * limit,
  });

  return (
    <div>
      <EmployeeTable
        employees={employees}
        currentPage={page}
      />
    </div>
  );
}
```

```typescript
// app/admin/karyawan/components/employee-filter.tsx — Client Component
// Filter interaktif yang update URL params (tidak menyentuh DB)

'use client';

import { useRouter, useSearchParams } from 'next/navigation';
import { Input } from '@/components/ui/input';
import { useCallback, useTransition } from 'react';

export function EmployeeFilter() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [isPending, startTransition] = useTransition();

  const handleSearch = useCallback((value: string) => {
    const params = new URLSearchParams(searchParams);
    if (value) params.set('search', value);
    else params.delete('search');
    params.delete('page');

    startTransition(() => {
      router.push(`?${params.toString()}`);
    });
  }, [router, searchParams]);

  return (
    <Input
      placeholder="Cari nama atau NIK..."
      defaultValue={searchParams.get('search') ?? ''}
      onChange={(e) => handleSearch(e.target.value)}
      disabled={isPending}
    />
  );
}
```

---

## 6. Authentication & Authorization

### 6.1 next-auth v5 Setup

```typescript
// lib/auth/config.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import { getDb } from '@/lib/db';
import { comparePassword } from '@/lib/utils/crypto';

export const { auth, handlers, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      credentials: {
        email: { type: 'email' },
        password: { type: 'password' },
        tenantSlug: { type: 'text' },
      },
      async authorize(credentials) {
        const sql = getDb();
        const rows = await sql`
          SELECT e.id, e.full_name, e.email, e.password_hash, e.role,
                 e.tenant_id, t.slug AS tenant_slug, e.is_active
          FROM employees e
          JOIN tenants t ON t.id = e.tenant_id
          WHERE e.email = ${credentials.email}
            AND t.slug = ${credentials.tenantSlug}
            AND e.deleted_at IS NULL
          LIMIT 1
        `;

        const user = rows[0];
        if (!user || !user.is_active) return null;

        const valid = await comparePassword(
          credentials.password as string,
          user.password_hash
        );
        if (!valid) return null;

        return {
          id: String(user.id),
          name: user.full_name,
          email: user.email,
          role: user.role,
          tenantId: String(user.tenant_id),
          tenantSlug: user.tenant_slug,
        };
      },
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.role = user.role;
        token.tenantId = user.tenantId;
        token.tenantSlug = user.tenantSlug;
      }
      return token;
    },
    session({ session, token }) {
      session.user.role = token.role as string;
      session.user.tenantId = token.tenantId as string;
      session.user.tenantSlug = token.tenantSlug as string;
      return session;
    },
  },
  pages: {
    signIn: '/login',
    error: '/login',
  },
});
```

### 6.2 Middleware Guard (Edge Runtime)

```typescript
// middleware.ts
import { auth } from '@/lib/auth/config';
import { NextResponse } from 'next/server';

const ROLE_ROUTES: Record<string, string[]> = {
  '/admin':    ['hr_admin', 'hr_staff', 'manager', 'supervisor'],
  '/employee': ['employee', 'supervisor', 'manager', 'hr_admin', 'hr_staff'],
};

export default auth((req) => {
  const { pathname } = req.nextUrl;
  const session = req.auth;

  if (!session && pathname !== '/login') {
    return NextResponse.redirect(new URL('/login', req.url));
  }

  for (const [prefix, roles] of Object.entries(ROLE_ROUTES)) {
    if (pathname.startsWith(prefix)) {
      if (!roles.includes(session?.user?.role ?? '')) {
        return NextResponse.redirect(new URL('/unauthorized', req.url));
      }
    }
  }

  return NextResponse.next();
});

export const config = {
  matcher: ['/((?!api/auth|_next/static|_next/image|favicon.ico).*)'],
};
```

### 6.3 Role Guard Helper

```typescript
// lib/auth/guards.ts
import { Session } from 'next-auth';
import { NextResponse } from 'next/server';

type Role = 'hr_admin' | 'hr_staff' | 'manager' | 'supervisor' | 'employee';

export function requireRole(session: Session, allowedRoles: Role[]) {
  if (!allowedRoles.includes(session.user.role as Role)) {
    throw new Response(
      JSON.stringify({ error: 'Forbidden' }),
      { status: 403, headers: { 'Content-Type': 'application/json' } }
    );
  }
}
```

---

### 6.4 Temuan Keamanan Prototipe (WAJIB diperbaiki sebelum data asli masuk)
Audit 24 Okt 2026 pada https://digihrd.vercel.app/:
- Aplikasi berjalan **"Mode HR Admin (Tanpa Login)"**; `/login` 404.
- Endpoint `/api/employees`, `/api/attendances`, `/api/payroll-runs`, `/api/locations`, dll. **bisa dibaca publik tanpa sesi** dan mengembalikan data pribadi (NIK, telepon, email) — saat ini masih data dummy.

Aksi:
1. Aktifkan `middleware.ts` (§6.2) untuk `/((?!api/auth|_next|favicon).*)`; hapus mode tanpa login (boleh hanya di `NODE_ENV=development` dengan flag eksplisit `DEV_AUTH_BYPASS=1`, tidak pernah di production).
2. Setiap route handler memanggil `requireSession()` + `requireRole()` + filter `tenant_id` (CLAUDE.md aturan #6).
3. Response DTO tidak menyertakan `nik`, `npwp`, `bank_account_number`, `phone`, `password_hash` kecuali endpoint detail khusus HR_ADMIN.
4. Hapus seed dummy (PT Maju, Pabrik Cimahi, dst.) dari database prototipe sebelum import data ABPS.
5. Test e2e: semua `/api/*` tanpa cookie → 401.

### 6.5 Login Karyawan Lapangan
Banyak crew tidak memiliki email kerja → login dengan `employee_number` + PIN 6 digit (credentials provider), email opsional. Rate limit 5 percobaan/15 menit.

---

## 7. Upload & File Storage

### 7.1 Vercel Blob Pattern

```typescript
// lib/blob/upload.ts
import { put, del } from '@vercel/blob';

export type BlobFolder =
  | 'attendance-photos'
  | 'payslips'
  | 'logos'
  | 'leave-documents'
  | 'exports';

export async function uploadFile(
  file: File | Blob,
  folder: BlobFolder,
  filename: string
): Promise<string> {
  const { url } = await put(`${folder}/${filename}`, file, {
    access: 'public',
    contentType: file instanceof File ? file.type : 'application/octet-stream',
  });
  return url;
}

export async function deleteFile(url: string): Promise<void> {
  await del(url);
}
```

```typescript
// app/api/upload/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authConfig } from '@/lib/auth/config';
import { uploadFile } from '@/lib/blob/upload';
import { nanoid } from 'nanoid';

export async function POST(req: NextRequest) {
  const session = await getServerSession(authConfig);
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const formData = await req.formData();
  const file = formData.get('file') as File | null;
  const folder = formData.get('folder') as string;

  if (!file) return NextResponse.json({ error: 'No file' }, { status: 400 });

  const ext = file.name.split('.').pop();
  const filename = `${nanoid()}.${ext}`;

  try {
    const url = await uploadFile(file, folder as any, filename);
    return NextResponse.json({ url });
  } catch (err) {
    console.error('[POST /api/upload]', err);
    return NextResponse.json({ error: 'Upload failed' }, { status: 500 });
  }
}
```

---

## 8. Fitur Khusus — Implementasi Teknis

### 8.1 Absensi Geolokasi

```typescript
// app/employee/absensi/components/clock-in-widget.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { toast } from 'sonner';
import { Navigation } from 'lucide-react';

const MOCK_GPS_ACCURACY_THRESHOLD = 5; // meter — suspiciously perfect

interface ClockInWidgetProps {
  assignedLocation: { lat: number; lng: number; radius: number; name: string };
  shiftAssignmentId: bigint;
}

export function ClockInWidget({ assignedLocation, shiftAssignmentId }: ClockInWidgetProps) {
  const [loading, setLoading] = useState(false);
  const [status, setStatus] = useState<'idle' | 'clocked-in'>('idle');

  async function handleClockIn() {
    setLoading(true);

    if (!navigator.geolocation) {
      toast.error('GPS tidak tersedia di perangkat ini.');
      setLoading(false);
      return;
    }

    navigator.geolocation.getCurrentPosition(
      async (position) => {
        const { latitude, longitude, accuracy } = position.coords;

        const payload = {
          lat: latitude,
          lng: longitude,
          accuracy,
          shiftAssignmentId: String(shiftAssignmentId),
        };

        const res = await fetch('/api/attendances', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload),
        });

        const data = await res.json();

        if (!res.ok) {
          toast.error(data.error ?? 'Gagal clock-in.');
        } else if (data.status === 'mock_gps') {
          toast.error('GPS palsu terdeteksi. Absensi ditolak.');
        } else if (data.status === 'outside_radius') {
          // Trigger modal luar radius — handled by parent
          toast.warning(`Anda berada ${data.distance}m dari lokasi. Di luar radius ${assignedLocation.radius}m.`);
        } else {
          setStatus('clocked-in');
          toast.success('Clock-in berhasil!');
        }
        setLoading(false);
      },
      (err) => {
        toast.error('Tidak dapat mengakses GPS. Aktifkan lokasi perangkat Anda.');
        setLoading(false);
      },
      { enableHighAccuracy: true, timeout: 10000, maximumAge: 0 }
    );
  }

  return (
    <Button onClick={handleClockIn} disabled={loading || status === 'clocked-in'}>
      <Navigation className="mr-2 h-4 w-4" />
      {loading ? 'Mendeteksi lokasi...' : status === 'clocked-in' ? 'Sudah Clock-in' : 'Clock In'}
    </Button>
  );
}
```

```typescript
// lib/geo/distance.ts

// Haversine formula — jarak dua koordinat GPS dalam meter
export function haversineDistance(
  lat1: number, lng1: number,
  lat2: number, lng2: number
): number {
  const R = 6371000; // Earth radius in meters
  const dLat = toRad(lat2 - lat1);
  const dLng = toRad(lng2 - lng1);
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLng / 2) ** 2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}

function toRad(deg: number) { return deg * (Math.PI / 180); }

// Heuristik deteksi mock GPS:
// Accuracy < 5m dan koordinat bulat sempurna = sangat mencurigakan
export function isSuspectedMockGps(
  accuracy: number,
  lat: number,
  lng: number
): boolean {
  if (accuracy !== null && accuracy < 3) return true;
  const latDecimals = String(lat).split('.')[1]?.length ?? 0;
  const lngDecimals = String(lng).split('.')[1]?.length ?? 0;
  if (latDecimals <= 2 && lngDecimals <= 2) return true;
  return false;
}
```

```typescript
// app/api/attendances/route.ts — POST clock-in handler

import { haversineDistance, isSuspectedMockGps } from '@/lib/geo/distance';
import { getActiveShiftAssignment, createAttendance } from '@/lib/db/queries/attendances';
import { getLocationById } from '@/lib/db/queries/locations';
import { getLateRule } from '@/lib/db/queries/payroll';

export async function POST(req: NextRequest) {
  const session = await getServerSession(authConfig);
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const { lat, lng, accuracy, shiftAssignmentId } = await req.json();

  const employeeId = BigInt(session.user.id);
  const tenantId = BigInt(session.user.tenantId);
  const now = new Date();

  // 1. Get assignment & location
  const assignment = await getActiveShiftAssignment(BigInt(shiftAssignmentId), employeeId);
  if (!assignment) return NextResponse.json({ error: 'Shift tidak ditemukan' }, { status: 404 });

  const location = await getLocationById(assignment.location_id, tenantId);
  if (!location) return NextResponse.json({ error: 'Lokasi tidak ditemukan' }, { status: 404 });

  // 2. Mock GPS check
  const isMock = isSuspectedMockGps(accuracy, lat, lng);

  // 3. Distance check
  const distance = haversineDistance(lat, lng, location.latitude, location.longitude);
  const isOutside = distance > location.radius_meter;

  // 4. Late calculation
  const shiftStart = new Date(`${now.toDateString()} ${assignment.start_time}`);
  const lateMinutes = Math.max(0, Math.floor((now.getTime() - shiftStart.getTime()) / 60000));

  // 5. Determine clock-in status
  let clockInStatus = 'valid';
  if (isMock) clockInStatus = 'mock_gps';
  else if (isOutside) clockInStatus = 'outside_radius';

  const attendanceStatus = isMock ? 'present' : lateMinutes > 0 ? 'late' : 'present';

  // 6. Create attendance record
  const record = await createAttendance({
    tenantId,
    employeeId,
    locationId: location.id,
    shiftAssignmentId: BigInt(shiftAssignmentId),
    attendanceDate: now.toISOString().split('T')[0],
    clockInAt: now,
    clockInLat: lat,
    clockInLng: lng,
    clockInAccuracy: accuracy,
    clockInDistance: Math.round(distance),
    clockInStatus,
    isMockGps: isMock,
    fraudFlag: isMock,
    lateMinutes,
    status: attendanceStatus,
  });

  return NextResponse.json({
    data: record,
    status: clockInStatus,
    distance: Math.round(distance),
    lateMinutes,
  });
}
```

### 8.2 Generate PDF Slip Gaji

```typescript
// lib/pdf/slip-template.tsx
import {
  Document, Page, Text, View, StyleSheet, Image, Font
} from '@react-pdf/renderer';

Font.register({
  family: 'Cabin',
  src: 'https://fonts.gstatic.com/s/cabin/v27/u-4i0b0a.ttf',
});

const styles = StyleSheet.create({
  page:    { fontFamily: 'Cabin', padding: 32, fontSize: 10, color: '#334155' },
  header:  { backgroundColor: '#1D4ED8', padding: 16, marginBottom: 16, borderRadius: 6 },
  // ... complete styles
});

interface SlipData {
  employee: { name: string; nik: string; position: string; bank: string; accountNo: string };
  period: string;
  earnings: { label: string; amount: number }[];
  deductions: { label: string; amount: number }[];
  attendance: { present: number; absent: number; late: number; hours: number };
  totalEarnings: number;
  totalDeductions: number;
  netSalary: number;
  brandColor: string;
  companyName: string;
}

export function SlipGajiDocument({ data }: { data: SlipData }) {
  return (
    <Document>
      <Page size="A5" style={styles.page}>
        {/* Header */}
        <View style={[styles.header, { backgroundColor: data.brandColor }]}>
          <Text style={{ color: 'white', fontSize: 14, fontWeight: 'bold' }}>
            SLIP GAJI KARYAWAN
          </Text>
          <Text style={{ color: 'rgba(255,255,255,0.8)', fontSize: 9, marginTop: 2 }}>
            Periode: {data.period}  ·  {data.companyName}
          </Text>
        </View>

        {/* Employee info */}
        <View style={{ marginBottom: 12 }}>
          {[
            ['Nama', data.employee.name],
            ['NIK', data.employee.nik],
            ['Jabatan', data.employee.position],
            ['Rekening', `${data.employee.bank} — ${data.employee.accountNo}`],
          ].map(([label, value]) => (
            <View key={label} style={{ flexDirection: 'row', marginBottom: 3 }}>
              <Text style={{ width: 80, color: '#64748B' }}>{label}</Text>
              <Text style={{ flex: 1 }}>: {value}</Text>
            </View>
          ))}
        </View>

        {/* Earnings & Deductions */}
        {/* ... complete template */}

        {/* Total */}
        <View style={{ backgroundColor: data.brandColor + '15', padding: 10, borderRadius: 4, flexDirection: 'row', justifyContent: 'space-between' }}>
          <Text style={{ fontWeight: 'bold', fontSize: 11 }}>TAKE HOME PAY</Text>
          <Text style={{ fontWeight: 'bold', fontSize: 13, color: data.brandColor }}>
            Rp {data.netSalary.toLocaleString('id-ID')}
          </Text>
        </View>
      </Page>
    </Document>
  );
}
```

```typescript
// app/api/payroll/items/[itemId]/slip/route.ts
import { renderToBuffer } from '@react-pdf/renderer';
import { SlipGajiDocument } from '@/lib/pdf/slip-template';
import { getPayrollItemDetail } from '@/lib/db/queries/payroll';
import { uploadFile } from '@/lib/blob/upload';

export async function POST(req: NextRequest, { params }: { params: { itemId: string } }) {
  const session = await getServerSession(authConfig);
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const item = await getPayrollItemDetail(BigInt(params.itemId), BigInt(session.user.tenantId));
  if (!item) return NextResponse.json({ error: 'Not found' }, { status: 404 });

  // Generate PDF buffer
  const buffer = await renderToBuffer(<SlipGajiDocument data={item} />);

  // Upload to Vercel Blob
  const filename = `slip-${item.employee.nik}-${item.period.replace(' ', '-')}.pdf`;
  const url = await uploadFile(
    new Blob([buffer], { type: 'application/pdf' }),
    'payslips',
    filename
  );

  // Save URL to DB
  await updatePayrollItemSlipUrl(BigInt(params.itemId), url);

  return NextResponse.json({ url });
}
```

### 8.3 Payroll Calculator ABPS (Pure Functions)

> v2.1: `PayRules` di bawah = `ResolvedRules` hasil `lib/rules/resolver.ts` (§15) yang dibekukan di `payroll_run_snapshots`. Setiap percabangan opsi (rounding, total-hours basis, alokasi bonus, dst.) dipanggil lewat `lib/rules/strategies/*`, bukan if/else hardcode.
>
> v2.0 menggantikan calculator generik v1.0 (multiplier lembur 1,5 + BPJS hardcode). Semua fungsi **murni** (tanpa DB), menerima `PayRules` dari tabel `pay_rules` versi yang berlaku pada `period_end`. **Dilarang** hardcode angka 26, 9, 500, 10.000, dst. di luar seed/test fixture.

Struktur file:
```
lib/payroll/
├── types.ts            # PayRules, DailyAttendance, PayrollLine, PayrollResult
├── money.ts            # pembulatan rupiah (Math.round ke rupiah) — satu-satunya tempat pembulatan
├── period.ts           # weeklyPeriod(date, weekStartDow=6) → {start: Sabtu, end: Jumat}; monthlyPeriod()
├── weekly-hourly.ts    # produksi pusat
├── freelance.ts        # tarif jam + borongan + kurir
├── monthly-split.ts    # crew outlet & kantor
├── part-time.ts        # fixed + visit
├── overtime.ts         # aturan 9 jam + grace 30 mnt + roundup
├── bonus.ts            # tier omset → pool → alokasi
├── deductions.ts       # telat, izin, sakit, prorate, kasbon, gantungan
├── bpjs.ts / pph21.ts  # DIPERTAHANKAN tapi hanya dipanggil jika tenant.feature_bpjs/pph21 = true (ABPS: false)
└── __tests__/          # fixture dari sheet asli (lihat 8.3.6)
```

#### 8.3.1 Lembur (pusat)
```typescript
// lib/payroll/overtime.ts
export function payableOvertimeHours(workedMinutes: number, r: PayRules): number {
  const over = workedMinutes - r.overtimeThresholdMinutes;          // 540
  if (over <= 0) return 0;
  if (over <= r.overtimeGraceMinutes) return 0;                    // ≤ 30 menit
  return Math.ceil((over - r.overtimeGraceMinutes) / 60);          // = ROUNDUP(...,0) di sheet
}
// Bulanan: jam lembur berasal dari overtime_requests.payable_hours (boleh 0,5) × r.overtimeFlatRate
```

#### 8.3.2 Mingguan produksi pusat
```typescript
// lib/payroll/weekly-hourly.ts
export function calcWeeklyDay(d: DailyAttendance, emp: { dailyRate: number }, r: PayRules) {
  const hourly = emp.dailyRate / r.standardHoursPerDay;                       // 130.000 / 9
  const countIn = r.earliestCountTime ? maxTime(d.clockIn, r.earliestCountTime) : d.clockIn; // MAX(masuk, 05:00)
  const minutes = diffMinutes(countIn, d.clockOut);                           // Q-04: pakai countIn (bukan jam asli)
  const base = d.isFull9hDay ? emp.dailyRate : (minutes / 60) * hourly;        // "Hari Kerja Full 9 Jam" → flat
  const otHours = payableOvertimeHours(minutes, r);
  return {
    base,
    overtime: otHours * r.overtimeFlatRate,                                    // 10.000/jam
    earlyIncentive: r.earlyArrivalBefore && d.clockIn < r.earlyArrivalBefore ? r.earlyArrivalIncentive : 0,
    holidayIncentive: d.holiday ? d.holiday.incentiveAmount : 0,               // 50.000
  };
}
// Total minggu = Σ hari + penambahan (bonus akhir tahun, jam Minggu terpisah) − kasbon
```

#### 8.3.3 Bulanan (crew outlet & kantor)
```typescript
// lib/payroll/monthly-split.ts
export function calcMonthly(i: MonthlyInput, r: PayRules): PayrollResult {
  const total = i.pokok + i.tunjTidakTetap + i.tunjJabatan;  // nominal dari employee_salary_components
  const perDay = total / r.divisorDaysPerMonth;              // ÷ 26
  // Perubahan gaji tengah bulan / karyawan baru: hitung per segmen effective-dated:
  //   Σ (gajiSegmen / 26 × hariKerjaSegmen)
  const late = r.lateDeductionMode === 'per_minute'
    ? i.lateMinutes * r.lateDeductionPerMinute                               // 500/menit
    : Math.round((i.lateMinutes / 60) * (total / r.divisorDaysPerMonth / r.standardHoursPerDay)); // kantor
  const izin  = i.izinDays  * basisPerDay(r.izinDeductionBasis, i, r);       // crew: total/26; kantor: tunjTT/26
  const sakit = r.sakitFlatPerDay != null ? i.sickDays * r.sakitFlatPerDay
                                          : i.sickDays * (i.tunjTidakTetap / r.divisorDaysPerMonth);
  const overtime = i.overtimeHours * r.overtimeFlatRate;                     // 10.000 crew / 17.000 kantor
  const earnings = total + overtime + i.bonus + i.transport + i.holidayIncentive + i.visitFee;
  const deductions = late + izin + sakit + i.absentProrate + i.kasbon + i.holdback;
  return build(earnings, deductions /* THP = earnings − deductions */);
}
```
Libur operasional (`holidays.is_closure`, mis. Lebaran) → hari tsb dibayar `closure_pay_percent` (50%) dari gaji harian.

#### 8.3.4 Freelance & part-time
`freelance.ts`: jam × 9.000 + Σ piece_work_entries.amount. `part-time.ts`: fixed + jumlah visit × rate_snapshot.

#### 8.3.5 Bonus omset outlet
```typescript
// lib/payroll/bonus.ts
export function bonusPool(netRevenue: number, tiers: Tier[]): { percent: number; pool: number } {
  const t = tiers.find(t => netRevenue >= t.min && (t.max == null || netRevenue <= t.max));
  const percent = t?.percent ?? 0;                                   // <350jt 0% | 350–400jt 0,5% | >400jt 1%
  return { percent, pool: Math.round(netRevenue * percent / 100) };   // net = kotor − diskon − ojol
}
// Alokasi: 'manual' (default, sesuai praktik Sept: 180rb/80rb), 'equal', 'attendance_weighted' — Q-02
```

#### 8.3.6 Test fixture wajib (dari sheet asli, disimpan tanpa PII)
| Fixture | Sumber | Ekspektasi |
|---|---|---|
| `weekly-2026-10-10.json` | Gaji Mingguan Okt (minggu Sabtu–Jumat) | upah/hari, lembur, insentif = sheet |
| `overtime-cases.json` | Rumus kolom lembur | 9j00→0; 9j30→0; 9j31→1; 10j31→2 |
| `monthly-2026-09-outlet.json` | Gaji Bulanan Sept | THP per crew = sheet (telat 500/mnt, izin total/26, sakit tunjTT/26) |
| `monthly-2026-09-kantor.json` | baris kantor | telat prorata, sakit 50rb, lembur 17rb |
| `prorate-raise.json` | Purchasing 2jt→2,3jt | Σ segmen /26 |
| `bonus-2026-09.json` | kolom BONUS | pool 1.760.000 @ 352jt × 0,5% (inferensi) |
Coverage `lib/payroll/**` ≥ 95%.

#### 8.3.7 Kasbon, Borongan, Holdback (alur data)
- `cash_advances` (status `open`) dibaca saat generate; potongan = MIN(sisa, cicilan ?? sisa, gaji tersedia). Saat run `approved` → buat `cash_advance_repayments`, kurangi `remaining_amount`; saat run dibatalkan → rollback dalam transaksi yang sama.
- `piece_work_entries` dalam periode → komponen `BORONGAN` (detail qty × rate).
- `salary_holdbacks` → komponen `GANTUNGAN` (potongan) saat ditahan; dilepas sebagai pendapatan pada run yang ditentukan HR.

#### 8.3.8 Payroll Run Lifecycle (mingguan & bulanan)
```
POST /api/payroll-runs            { payGroupId, periodStart }   → hitung periodEnd dari pay_groups
POST /api/payroll-runs/:id/generate                              → draft → calculated (transaksi, idempotent: hapus item lama)
POST /api/payroll-runs/:id/review | /approve | /mark-paid        → role check (PRD §4)
GET  /api/payroll-runs/:id/items/:itemId/slip                     → PDF (template mingguan/bulanan)
GET  /api/reports/payroll?runId=&format=xlsx                      → kolom mengikuti sheet ABPS; BPJS/PPh21 disembunyikan bila non-aktif
```
Unik: `(tenant_id, pay_group_id, period_start)`. Run `approved/paid` immutable; koreksi = run tambahan (`adjustment`).


### 8.4 Excel Export

```typescript
// lib/excel/generators.ts
import * as XLSX from 'xlsx';

export function generateAttendanceExcel(
  data: AttendanceRow[],
  period: string
): Buffer {
  const wb = XLSX.utils.book_new();

  const rows = data.map(r => ({
    'NIK':             r.nik,
    'Nama':            r.full_name,
    'Departemen':      r.department,
    'Tanggal':         r.attendance_date,
    'Jam Masuk':       r.clock_in ?? '-',
    'Jam Keluar':      r.clock_out ?? '-',
    'Status GPS':      r.clock_in_status,
    'Jarak (m)':       r.clock_in_distance ?? '-',
    'Terlambat (mnt)': r.late_minutes,
    'Jam Kerja':       r.effective_work_minutes ? (r.effective_work_minutes / 60).toFixed(2) : '-',
    'Status':          r.status,
  }));

  const ws = XLSX.utils.json_to_sheet(rows);

  // Column widths
  ws['!cols'] = [
    { wch: 16 }, { wch: 24 }, { wch: 18 }, { wch: 12 },
    { wch: 10 }, { wch: 10 }, { wch: 14 }, { wch: 10 },
    { wch: 16 }, { wch: 10 }, { wch: 12 },
  ];

  XLSX.utils.book_append_sheet(wb, ws, `Kehadiran ${period}`);
  return XLSX.write(wb, { type: 'buffer', bookType: 'xlsx' });
}
```

```typescript
// app/api/reports/attendance/route.ts
export async function GET(req: NextRequest) {
  // ...session check...
  const format = req.nextUrl.searchParams.get('format') ?? 'json';

  const data = await getAttendanceReport({ tenantId, ...filters });

  if (format === 'xlsx') {
    const buffer = generateAttendanceExcel(data, period);
    return new NextResponse(buffer, {
      headers: {
        'Content-Type': 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'Content-Disposition': `attachment; filename="kehadiran-${period}.xlsx"`,
      },
    });
  }

  return NextResponse.json({ data });
}
```

---

## 9. Performance & Optimasi

### 9.1 Next.js Caching Strategy

```typescript
// Revalidate secara selektif per route
// app/admin/karyawan/page.tsx
export const revalidate = 30; // ISR: revalidate setiap 30 detik untuk data yang jarang berubah

// app/admin/absensi/page.tsx
export const dynamic = 'force-dynamic'; // Tidak cache — data kehadiran realtime

// app/admin/payroll/page.tsx
export const revalidate = 60; // Payroll bisa di-cache lebih lama
```

### 9.2 Query Optimization Rules

- Semua query yang dilakukan di high-traffic routes (absensi hari ini, dashboard) **harus menggunakan index yang sudah didefinisikan di ERD**.
- Setiap query list yang bisa mengembalikan banyak baris **wajib menggunakan LIMIT + OFFSET** atau cursor-based pagination.
- Aggregation query (total kehadiran, total payroll) **menggunakan CTE** untuk readability dan performa.
- Query yang melibatkan rentang tanggal **wajib menggunakan index pada kolom tanggal** — tidak boleh menggunakan `DATE()` function yang membuat index tidak terpakai.

```sql
-- ❌ BURUK — tidak menggunakan index
WHERE DATE(clock_in_at) = '2026-10-11'

-- ✅ BAIK — menggunakan index pada timestamptz
WHERE clock_in_at >= '2026-10-11 00:00:00+07'
  AND clock_in_at <  '2026-10-12 00:00:00+07'
```

### 9.3 Komponen & Lazy Loading

```typescript
// Komponen berat di-lazy load
import dynamic from 'next/dynamic';

// Chart hanya load di client ketika visible
const AttendanceBarChart = dynamic(
  () => import('@/components/charts/attendance-bar'),
  { ssr: false, loading: () => <ChartSkeleton /> }
);

// PDF renderer tidak perlu SSR
const SlipPdfViewer = dynamic(
  () => import('@/app/admin/slip-gaji/components/slip-pdf'),
  { ssr: false }
);

// Shift scheduler dengan dnd-kit
const ShiftScheduler = dynamic(
  () => import('@/app/admin/shift/components/shift-scheduler'),
  { ssr: false, loading: () => <SchedulerSkeleton /> }
);
```

### 9.4 Image Optimization

```typescript
// Semua gambar dari Vercel Blob menggunakan next/image
import Image from 'next/image';

// next.config.ts
const config = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '*.public.blob.vercel-storage.com',
      },
    ],
  },
};
```

---

## 10. Environment Variables

```bash
# .env.local (development) — .env.production (production di Vercel dashboard)

# Database
DATABASE_URL=postgresql://user:pass@ep-xxx.neon.tech/hrd?sslmode=require

# Auth
NEXTAUTH_SECRET=<random-32-char-string>
NEXTAUTH_URL=http://localhost:3000   # production: https://yourdomain.vercel.app

# Storage
BLOB_READ_WRITE_TOKEN=vercel_blob_rw_...

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 11. Error Handling & Logging

### 11.1 API Route Error Pattern

```typescript
// Semua API route menggunakan pattern yang konsisten
try {
  // ... operasi
} catch (err) {
  // Log dengan konteks yang cukup untuk debugging
  console.error(`[${req.method} ${req.nextUrl.pathname}]`, {
    error: err instanceof Error ? err.message : err,
    tenantId: session?.user?.tenantId,
    userId: session?.user?.id,
  });

  // Jangan expose detail error ke client di production
  if (err instanceof z.ZodError) {
    return NextResponse.json({ error: err.flatten() }, { status: 422 });
  }
  return NextResponse.json(
    { error: process.env.NODE_ENV === 'development' ? String(err) : 'Internal server error' },
    { status: 500 }
  );
}
```

### 11.2 Client Error Boundary

```typescript
// app/(dashboard)/error.tsx
'use client';
export default function DashboardError({
  error, reset,
}: { error: Error; reset: () => void }) {
  return (
    <div className="flex flex-col items-center justify-center h-full gap-4">
      <p className="text-muted-foreground">Terjadi kesalahan. Coba muat ulang halaman.</p>
      <Button onClick={reset}>Muat Ulang</Button>
    </div>
  );
}
```

---

## 12. Deployment & CI/CD

### 12.1 Vercel Configuration

```json
// vercel.json
{
  "buildCommand": "npm run build",
  "installCommand": "npm ci",
  "framework": "nextjs",
  "regions": ["sin1"],
  "env": {
    "NODE_ENV": "production"
  }
}
```

### 12.2 GitHub Actions (Pre-deploy checks)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run type-check    # tsc --noEmit
      - run: npm run lint          # eslint
      - run: npm run test          # vitest
```

### 12.3 Branch Strategy

```
main        → production (auto-deploy ke Vercel)
staging     → preview deployment (URL unik Vercel)
feature/*   → PR → staging → main
```

### 12.4 Database Migration Workflow

```bash
# Saat ada perubahan schema:
1. Edit drizzle/schema.ts
2. npx drizzle-kit generate    # buat file migration SQL
3. Review SQL yang di-generate
4. npx drizzle-kit migrate     # apply ke Neon database
5. Commit schema.ts + migration files
```

---

## 13. Standar Kode

### 13.1 Penamaan

| Entitas | Konvensi | Contoh |
|---|---|---|
| File komponen | kebab-case | `employee-table.tsx` |
| File query | kebab-case | `attendances.ts` |
| Komponen React | PascalCase | `EmployeeTable` |
| Fungsi query | camelCase + verb | `getEmployeeList`, `createAttendance` |
| API route | sesuai direktori | `/api/employees` → `GET`, `POST` |
| Type/Interface | PascalCase | `Employee`, `PayrollItem` |
| Env variable | SCREAMING_SNAKE_CASE | `DATABASE_URL` |
| CSS class | Tailwind utility only | tidak ada custom class kecuali `cn()` merge |

### 13.2 TypeScript Rules

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}

// DILARANG:
const x: any = ...            // ❌ gunakan unknown atau type yang tepat
// @ts-ignore                 // ❌ fix the error, don't ignore it
let result = await sql(...)   // ❌ always type the return: sql<MyType[]>

// WAJIB:
const result = await sql<Employee[]>`SELECT ...`  // ✅
```

### 13.3 Komentar Wajib di Query Kompleks

```typescript
// lib/db/queries/payroll.ts

// Ambil semua komponen gaji aktif untuk seorang karyawan
// pada periode tertentu, beserta nilai yang sudah di-override per karyawan.
// Jika ada override di employee_salary_components, gunakan itu.
// Jika tidak ada, gunakan default dari salary_components master.
export async function getEmployeeSalaryComponents(
  employeeId: bigint,
  referenceDate: string
): Promise<SalaryComponent[]> {
  const sql = getDb();
  return sql<SalaryComponent[]>`
    SELECT
      sc.id,
      sc.name,
      sc.code,
      sc.component_type,
      sc.is_taxable,
      sc.is_bpjs_base,
      sc.calculation_basis,
      COALESCE(esc.amount, sc.default_amount) AS amount,
      COALESCE(esc.rate,   sc.default_rate)   AS rate
    FROM salary_components sc
    LEFT JOIN employee_salary_components esc
      ON esc.salary_component_id = sc.id
     AND esc.employee_id = ${employeeId}
     AND esc.effective_from <= ${referenceDate}
     AND (esc.effective_until IS NULL OR esc.effective_until >= ${referenceDate})
    WHERE sc.tenant_id = (
      SELECT tenant_id FROM employees WHERE id = ${employeeId}
    )
    AND sc.is_active = TRUE
    ORDER BY sc.component_type, sc.name
  `;
}
```

---

## 14. Dependency Version Lock

```json
// package.json — versi yang digunakan, jangan upgrade tanpa diskusi tim
{
  "dependencies": {
    "next":                      "15.x",
    "react":                     "19.x",
    "react-dom":                 "19.x",
    "typescript":                "5.x",
    "@neondatabase/serverless":  "0.10.x",
    "next-auth":                 "5.x",
    "drizzle-orm":               "0.38.x",
    "zod":                       "3.x",
    "lucide-react":              "0.469.x",
    "tailwindcss":               "4.x",
    "@react-pdf/renderer":       "4.x",
    "xlsx":                      "0.18.x",
    "@dnd-kit/core":             "6.x",
    "@dnd-kit/sortable":         "8.x",
    "recharts":                  "2.x",
    "@tiptap/react":             "2.x",
    "@tiptap/starter-kit":       "2.x",
    "@vercel/blob":              "0.27.x",
    "nanoid":                    "5.x",
    "bcryptjs":                  "2.x",
    "sonner":                    "1.x"
  },
  "devDependencies": {
    "drizzle-kit":               "0.30.x",
    "vitest":                    "2.x",
    "@playwright/test":          "1.x",
    "eslint":                    "9.x",
    "prettier":                  "3.x",
    "husky":                     "9.x",
    "lint-staged":               "15.x"
  }
}
```

---

## 15. Rule Engine & Resolver Pengaturan (v2.1)

### 15.1 Prinsip
1. **Tidak ada nilai aturan di kode.** Calculator menerima objek `ResolvedRules`; ia tidak membaca DB dan tidak punya default sendiri.
2. **Satu pintu:** semua pembacaan setting lewat `lib/rules/resolver.ts`. Query langsung ke `setting_values` di luar `lib/db/queries/settings.ts` dilarang.
3. **Calculator berbasis strategi:** setiap opsi enum dipetakan ke fungsi strategi (`lib/rules/strategies/*`). Menambah opsi baru = menambah strategi + `setting_options` + test, tanpa mengubah calculator inti.
4. **Snapshot:** payroll run memakai snapshot beku, bukan setting live.

### 15.2 Struktur file
```
lib/rules/
├── registry.ts          # tipe key → TS type (di-generate dari setting_definitions: `pnpm rules:gen`)
├── resolver.ts          # resolveRules(ctx) / resolveOne(key, ctx)
├── validate.ts          # zod dari json_schema + validasi lintas setting
├── snapshot.ts          # buildSnapshot(run) / loadSnapshot(run)
└── strategies/
    ├── bonus-allocation.ts   # equal | attendance_days | position_weight | attendance_x_weight | manual | fixed_amount_with_penalty
    ├── tier-match.ts         # inclusive/exclusive boundaries
    ├── overtime-rounding.ts  # none | ceil_hour | floor_half | nearest_half | ceil_half
    ├── total-hours.ts        # counted_clock_in | raw_clock_in
    ├── late-deduction.ts     # per_minute | prorata_hourly | per_hour | flat_cut | tiered | none
    ├── leave-deduction.ts    # total_div_days | variable_allowance_div_days | base_div_days | flat | none
    ├── proration.ts          # by_divisor_days | by_calendar_days | by_working_days | none
    ├── sunday-policy.ts      # normal_hourly | separate_hourly_rate | multiplier | flat_per_day | excluded
    ├── holdback-release.ts   # manual | after_n_runs | on_resign | date
    └── revenue-net.ts        # komponen pengurang on/off
```

### 15.3 Resolver
```typescript
// lib/rules/resolver.ts
export interface RuleContext {
  tenantId: number; date: string;              // tanggal acuan (period_end / attendance_date)
  payGroupId?: number; locationId?: number; positionId?: number; employeeId?: number;
}
const ORDER = ['employee', 'position', 'location', 'pay_group', 'tenant'] as const;

export async function resolveRules(ctx: RuleContext, keys?: RuleKey[]): Promise<ResolvedRules> {
  const rows = await getActiveSettingValues(ctx, keys);   // lib/db/queries/settings.ts, 1 query (lihat 15.4)
  const defs = await getSettingDefinitions(keys);         // cache per request
  const out = {} as ResolvedRules;
  for (const def of defs) {
    const hit = ORDER.map(s => rows.find(r => r.key === def.key && r.scopeType === s)).find(Boolean);
    out[def.key] = { v: hit ? hit.value.v : def.defaultValue.v,
                     source: hit ? `${hit.scopeType}:${hit.scopeId ?? ''}` : 'default',
                     settingValueId: hit?.id ?? null };
  }
  return applyDependencies(out, defs);                    // child dimatikan jika parent toggle off
}
```

### 15.4 Query (raw SQL, satu round-trip)
```sql
-- getActiveSettingValues: ambil versi berlaku untuk semua scope yang relevan
SELECT sd.key, sv.id, sv.scope_type, sv.scope_id, sv.value
FROM setting_values sv
JOIN setting_definitions sd ON sd.id = sv.definition_id
WHERE sv.tenant_id = $1 AND sv.status = 'active'
  AND sv.effective_from <= $2 AND (sv.effective_to IS NULL OR sv.effective_to > $2)
  AND ( sv.scope_type = 'tenant'
     OR (sv.scope_type = 'pay_group' AND sv.scope_id = $3)
     OR (sv.scope_type = 'location'  AND sv.scope_id = $4)
     OR (sv.scope_type = 'position'  AND sv.scope_id = $5)
     OR (sv.scope_type = 'employee'  AND sv.scope_id = $6) )
  AND ($7::text[] IS NULL OR sd.key = ANY($7));
```
Untuk generate payroll, panggil sekali per run dengan semua employee (`scope_id = ANY(...)`) lalu resolve di memori (hindari N+1).

### 15.5 Validasi saat simpan (`POST /api/settings/values`)
1. Zod dari `json_schema`/`value_type`; `min_value`/`max_value`; enum harus ada di `setting_options.is_active`.
2. Scope harus termasuk `allowed_scopes`; `is_locked` → 403.
3. Tidak overlap (constraint `ex_no_overlap`). Versi baru otomatis menutup versi lama: `effective_to = new.effective_from`, status `superseded` dalam transaksi yang sama.
4. **Validasi lintas setting**: tier bonus tidak boleh tumpang tindih/bercelah (sesuai inclusive/exclusive); persen split gaji = 100; `threshold > 0`; jika `allocation_method = position_weight` maka bobot jabatan wajib ada; jika `revenue.source` berisi import maka mapping kolom tersedia.
5. **Larangan retroaktif ke run terkunci**: jika `effective_from <= period_end` sebuah run berstatus `approved/paid` → tolak, kecuali user memilih "buat run adjustment" (permission `payroll.adjust`).
6. **(v2.2)** `resolveApprovalPolicy(key, scope)` (§16.1) menentukan perlu persetujuan atau tidak, beserta role, jumlah, dan mode. Kolom statis `requires_approval` hanya fallback. Pembuat ≠ penyetuju.
7. Setiap aksi → `setting_audit_logs` (old/new value, rentang, alasan wajib, actor, IP) dalam transaksi yang sama.
8. Endpoint **preview**: `POST /api/settings/preview` menghitung ulang run `draft/calculated` dengan nilai usulan dan menampilkan selisih per karyawan, tanpa menyimpan.

### 15.6 Snapshot & immutability
- `generate` (draft → calculated): `resolveRules` per karyawan → simpan `payroll_run_snapshots` (resolved_settings + domain_snapshot: tier, tarif borongan, holidays, komponen gaji, bobot jabatan) + `settings_hash` + `engine_version`. Calculator **hanya** membaca snapshot.
- Re-generate saat `calculated/reviewed` → snapshot dibuat ulang; jika hash berubah, UI menampilkan "Pengaturan berubah sejak kalkulasi terakhir".
- `approve` → `settings_frozen_at = NOW()`; snapshot & item tidak bisa di-update (trigger `BEFORE UPDATE` menolak jika run approved/paid).
- Slip PDF dan laporan run lama dibangun dari snapshot, bukan dari setting live.
- Koreksi setelah approve → `run_type = 'adjustment'`, `adjusts_run_id`, selisih saja.

### 15.7 API Pengaturan
```
GET    /api/settings/definitions?category=          # katalog + opsi
GET    /api/settings/resolved?employeeId=&date=     # nilai efektif + sumbernya (untuk layar "kenapa angka ini?")
GET    /api/settings/values?key=&scope=             # riwayat versi
POST   /api/settings/values                         # buat versi baru (validasi 15.5)
POST   /api/settings/values/:id/approve | /reject
POST   /api/settings/preview                        # simulasi dampak
GET    /api/settings/audit?key=&from=&to=
```
Semua route: `requireSession` + permission dari `setting_definitions.edit_permission` + filter `tenant_id`.

### 15.8 Test wajib
- Resolver: urutan override 5 level, batas `effective_from/to`, dependency toggle.
- Setiap strategi: test per opsi (mis. bonus allocation dengan data Sept: manual = 180rb/80rb; equal = 1.760.000/12).
- Snapshot: ubah setting setelah approve → hasil slip tidak berubah.
- Default seed menghasilkan angka = fixture sheet (TRD §8.3.6).

---

## 16. Approval Policy Engine (v2.2)

### 16.1 Resolver kebijakan persetujuan
```typescript
// lib/rules/approval.ts
export async function resolveApprovalPolicy(defKey: string, ctx: RuleContext): Promise<ApprovalPolicy> {
  const def = await getSettingDefinition(defKey);
  if (def.isLocked) throw new ForbiddenError('SETTING_LOCKED');          // system.public_api, meta.*
  // urutan: definition+scope (employee→position→location→pay_group→tenant) → category+scope → seed default
  const p = await getActiveApprovalPolicy(ctx.tenantId, def.id, def.category, ctx, todayJakarta());
  return p ?? { requiresApproval: def.requiresApproval, approverRoles: ['owner'], minApprovers: 1,
                mode: 'any', source: 'definition_default' };
}
```
Scope yang dipakai = scope dari setting value yang diubah. Tanggal acuan = hari pengajuan (bukan effective_from value), agar tidak bisa "menghindari" approval dengan memilih tanggal.

### 16.2 Alur simpan setting (memperluas §15.5)
1. Validasi value (§15.5 langkah 1–5).
2. `policy = resolveApprovalPolicy(key, scope)`.
3. `requiresApproval = false` → value `active` + audit `create`.
   `true` → value `pending_approval` + `approval_requests` (dengan `policy_snapshot`) + audit `request` + notifikasi ke `approverRoles`.
4. `POST /api/approvals/:id/decide` → cek: approver punya salah satu `approverRoles`, approver ≠ requester, belum memutus. Simpan `approval_decisions` + audit.
   - `mode any`: approve jika jumlah approve ≥ `minApprovers`.
   - `mode each_role`: approve jika setiap role punya ≥ 1 approve dan total ≥ `minApprovers`.
   - Satu reject → `rejected` (konfigurabel kelak).
   Setelah approved, value diaktifkan dan versi lama di-supersede dalam satu transaksi.
5. Job kedaluwarsa: request melewati `expires_at` → `expired` + audit.

### 16.3 Mengubah kebijakan persetujuan (meta)
`POST /api/settings/approval-policies` (permission `settings.approval_policy.manage`, default Owner):
- Selalu membuat `approval_requests` target `approval_policy`, memakai policy terkunci `meta.approval_policy` (Owner + HR Admin, min 2, each_role). Jika hanya ada 1 Owner: Owner + HR Admin.
- **Floor (G3):** jika key `approval_floor` dan perubahan menonaktifkan approval / menurunkan `minApprovers` / mengurangi role → wajib 2 penyetuju, termasuk Owner.
- `effective_from >= today` (G4); tidak boleh overlap (constraint).
- Validasi: `approverRoles` harus role yang ada dan memiliki minimal `minApprovers` orang aktif selain pengaju (cegah kebuntuan); `allowSelfApproval` selalu false.
- Audit `policy_create`/`policy_supersede` dengan old/new policy, alasan wajib, notifikasi (G7).

### 16.4 Snapshot & payroll
Snapshot run (§15.6) juga menyimpan `approval_request_id` setiap setting value yang dipakai. Run tidak bisa di-generate jika ada value berstatus `pending_approval` di periode itu kecuali HR memilih "pakai versi aktif sebelumnya" (tercatat di snapshot).

### 16.5 API
```
GET  /api/settings/approval-policies?key=&scope=      # efektif + riwayat
POST /api/settings/approval-policies                  # buat versi (selalu lewat meta-approval)
GET  /api/approvals?status=pending&mine=1
POST /api/approvals/:id/decide   { decision, note }
POST /api/approvals/:id/cancel   (hanya requester, saat pending)
```

### 16.6 Test wajib
- Override policy per scope & tanggal; fallback ke default definisi.
- Self-approval ditolak; `each_role` vs `any`; kebuntuan (role tanpa orang) ditolak.
- Menonaktifkan approval BPJS butuh 2 penyetuju termasuk Owner; tercatat di audit.
- `system.public_api` & `meta.approval_policy` → 403 di semua endpoint.
- Request lama tetap memakai `policy_snapshot` walau policy berubah.
