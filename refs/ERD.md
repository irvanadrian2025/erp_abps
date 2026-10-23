# Entity Relationship Diagram
## Sistem HRD — Absensi & Payroll · Ayam Bakar Pucuk Singkong (ABPS)
**Database:** PostgreSQL 15+ (Neon) + PostGIS
**Versi:** 2.2 (v2.1 + aturan persetujuan dinamis, 24 Okt 2026)
**Sumber kebenaran aturan bisnis:** `PRD.md` §8 (Payroll ABPS). Perubahan dari v1.0 dicatat di `CHANGELOG-analisa.md`.

> **v2.2:** Syarat persetujuan kini dinamis lewat `approval_policies` (berversi, per scope), `approval_requests`/`approval_decisions` (multi-penyetuju), meta-approval Owner, floor, dan audit. `setting_definitions.requires_approval` hanya default seed. Lihat bagian **v2.2**.
>
> **v2.1:** Semua angka, toggle, dan metode aturan dipindah ke registry pengaturan dinamis: `setting_definitions`, `setting_options`, `setting_values` (berversi tanggal, per scope dengan urutan override), `setting_audit_logs`, dan `payroll_run_snapshots`. Lihat bagian **v2.1** di akhir skema. Nilai di `pay_rules` dan komentar "default" di bawah = **default awal**, bukan keputusan final.
>
> **Ringkasan perubahan v2.0**
> - Penggajian **dua frekuensi**: *mingguan* (Dapur Pusat, Sabtu–Jumat, upah per jam = gaji harian ÷ 9) dan *bulanan* (Outlet & Kantor Pusat, basis 26 hari).
> - `payroll_runs` tidak lagi unik per (tahun, bulan); diganti `pay_group` + `period_start`/`period_end`.
> - Lembur ABPS = **tarif flat per jam** (bukan multiplier) → `overtime_requests.rate_mode`.
> - Tabel baru: `pay_groups`, `pay_rules`, `holidays`, `cash_advances` (kasbon), `piece_rate_items` & `piece_work_entries` (borongan/kurir/metik), `outlet_revenues`, `revenue_bonus_tiers`, `bonus_pools` & `bonus_allocations`, `salary_holdbacks` (gaji gantungan), `visit_logs` (part-time per kunjungan).
> - BPJS/PPh21 tetap ada di skema tetapi **non-aktif default** untuk tenant ABPS (tidak ada bukti potongan di slip lapangan — lihat pertanyaan terbuka).
> - Seed dummy "PT Maju Bersama / CV Cahaya" diganti seed ABPS (tanpa data pribadi: NIK/KK/rekening TIDAK di-seed).

---

## Konvensi

- PK: `BIGSERIAL` (bukan UUID) untuk performa join & index di data ratusan ribu baris
- Semua kolom status/type: `VARCHAR(50)` (bukan ENUM) agar mudah migrasi dan extensible
- Soft delete: kolom `deleted_at TIMESTAMPTZ` pada tabel utama
- Timestamps: semua `TIMESTAMPTZ` (timezone-aware)
- Index dirancang untuk query high-traffic: absensi harian, payroll run, laporan rentang tanggal

---
## Diagram Relasi (ASCII)

```
tenants (1)──────────────────────────────────────────────────────────┐
    ├──(N) locations  (Outlet Baros, Dapur Pusat)                   │
    ├──(N) departments ──(N) positions ──(1) employment_grades       │
    ├──(N) pay_groups  (WEEKLY_PUSAT, MONTHLY_OUTLET, MONTHLY_KANTOR, PART_TIME)
    │        └──(N) pay_rules (versi per effective_from)             │
    ├──(N) holidays (tanggal merah + nominal insentif)               │
    ├──(N) employees ───────────────────────────────────────────────┤
    │        ├──(N) employee_salary_components (pokok/kehadiran/jabatan, effective-dated)
    │        ├──(N) attendances ──(0..1) overtime_requests           │
    │        ├──(N) leave_requests (izin / sakit / off)              │
    │        ├──(N) shift_assignments ──(1) shift_templates          │
    │        ├──(N) cash_advances (kasbon)                           │
    │        ├──(N) piece_work_entries ──(1) piece_rate_items        │
    │        ├──(N) visit_logs (part-time)                           │
    │        ├──(N) salary_holdbacks (gaji gantungan)                │
    │        └──(N) payroll_items ──(N) payroll_item_details         │
    ├──(N) outlet_revenues (omset kotor, diskon, potongan ojol, omset bersih)
    │        └──(1) bonus_pools ──(N) bonus_allocations              │
    ├──(N) revenue_bonus_tiers                                       │
    ├──(N) salary_components (master)                                │
    ├──(N) payroll_runs (pay_group + period_start..period_end)       │
    └──(N) late_rules (per grade / pay_group)                        │
```

---
## Schema SQL Lengkap

```sql
-- ============================================================
-- 0. EXTENSIONS
-- ============================================================
CREATE EXTENSION IF NOT EXISTS postgis;          -- geolocation ops
CREATE EXTENSION IF NOT EXISTS pg_trgm;          -- fulltext search nama karyawan

-- ============================================================
-- 1. TENANTS (Multi-tenancy)
-- ============================================================
CREATE TABLE tenants (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(255)    NOT NULL,
    slug                VARCHAR(100)    NOT NULL UNIQUE,
    logo_url            TEXT,
    primary_color       VARCHAR(7)      NOT NULL DEFAULT '#1A56DB',
    secondary_color     VARCHAR(7)      NOT NULL DEFAULT '#7E3AF2',
    font_heading        VARCHAR(50)     NOT NULL DEFAULT 'Cabin',
    font_body           VARCHAR(50)     NOT NULL DEFAULT 'Albert Sans',
    slip_header_color   VARCHAR(7)      NOT NULL DEFAULT '#1A56DB',
    timezone            VARCHAR(50)     NOT NULL DEFAULT 'Asia/Jakarta',
    currency            VARCHAR(10)     NOT NULL DEFAULT 'IDR',
    is_active           BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_tenants_slug ON tenants(slug) WHERE deleted_at IS NULL;

-- ============================================================
-- 2. LOCATIONS (Kantor / Outlet / Pabrik)
-- ============================================================
CREATE TABLE locations (
    id                          BIGSERIAL PRIMARY KEY,
    tenant_id                   BIGINT          NOT NULL REFERENCES tenants(id),
    name                        VARCHAR(255)    NOT NULL,
    address                     TEXT,
    latitude                    NUMERIC(10,7)   NOT NULL,
    longitude                   NUMERIC(10,7)   NOT NULL,
    geom                        GEOMETRY(Point, 4326),   -- PostGIS point
    radius_meter                INT             NOT NULL DEFAULT 100,
    allow_outside_with_reason   BOOLEAN         NOT NULL DEFAULT FALSE,
    max_outside_tolerance_count INT             NOT NULL DEFAULT 0,
    require_photo               BOOLEAN         NOT NULL DEFAULT FALSE,
    restrict_mock_location      BOOLEAN         NOT NULL DEFAULT TRUE,
    is_active                   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at                  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at                  TIMESTAMPTZ
);

CREATE INDEX idx_locations_tenant ON locations(tenant_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_locations_geom ON locations USING GIST(geom);

-- ============================================================
-- 3. DEPARTMENTS
-- ============================================================
CREATE TABLE departments (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   BIGINT          NOT NULL REFERENCES tenants(id),
    name        VARCHAR(255)    NOT NULL,
    code        VARCHAR(50),
    parent_id   BIGINT          REFERENCES departments(id),
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ
);

CREATE INDEX idx_departments_tenant ON departments(tenant_id) WHERE deleted_at IS NULL;

-- ============================================================
-- 4. EMPLOYMENT GRADES (Golongan/Level)
-- ============================================================
CREATE TABLE employment_grades (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   BIGINT          NOT NULL REFERENCES tenants(id),
    name        VARCHAR(100)    NOT NULL,  -- e.g. "Produksi", "Staf", "Manajemen"
    level       SMALLINT        NOT NULL DEFAULT 1,
    created_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_grades_tenant ON employment_grades(tenant_id);

-- ============================================================
-- 5. LATE RULES (per Grade)
-- ============================================================
CREATE TABLE late_rules (
    id                          BIGSERIAL PRIMARY KEY,
    tenant_id                   BIGINT          NOT NULL REFERENCES tenants(id),
    employment_grade_id         BIGINT          NOT NULL REFERENCES employment_grades(id),
    tolerance_minutes           INT             NOT NULL DEFAULT 0,
    late_affects_payroll        BOOLEAN         NOT NULL DEFAULT FALSE,
    deduction_method            VARCHAR(20)     NOT NULL DEFAULT 'none',
      -- values: 'none', 'per_minute', 'per_hour', 'flat_cut'
    deduction_amount            NUMERIC(15,2)   NOT NULL DEFAULT 0,
    deduction_rate              NUMERIC(8,4)    NOT NULL DEFAULT 0,
    max_late_count_per_month    INT             NOT NULL DEFAULT 99,
    late_count_triggers_warning INT             NOT NULL DEFAULT 3,
    effective_from              DATE            NOT NULL,
    effective_until             DATE,
    created_at                  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_late_rules_grade ON late_rules(employment_grade_id);
CREATE INDEX idx_late_rules_tenant ON late_rules(tenant_id);

-- ============================================================
-- 6. POSITIONS (Jabatan)
-- ============================================================
CREATE TABLE positions (
    id                      BIGSERIAL PRIMARY KEY,
    tenant_id               BIGINT          NOT NULL REFERENCES tenants(id),
    department_id           BIGINT          NOT NULL REFERENCES departments(id),
    employment_grade_id     BIGINT          NOT NULL REFERENCES employment_grades(id),
    name                    VARCHAR(255)    NOT NULL,
    code                    VARCHAR(50),
    payroll_type            VARCHAR(20)     NOT NULL DEFAULT 'monthly',
      -- values: 'monthly_split' (bulanan pokok/tunjangan/jabatan), 'weekly_hourly' (harian ÷ 9, mingguan),
      --         'freelance_hourly' (tarif jam + borongan), 'part_time_visit' (fixed + per kunjungan)
    is_active               BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at              TIMESTAMPTZ
);

CREATE INDEX idx_positions_tenant ON positions(tenant_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_positions_dept ON positions(department_id);
CREATE INDEX idx_positions_grade ON positions(employment_grade_id);

-- ============================================================
-- 7. EMPLOYEES
-- ============================================================
CREATE TABLE employees (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    position_id         BIGINT          NOT NULL REFERENCES positions(id),
    location_id         BIGINT          REFERENCES locations(id),
    pay_group_id        BIGINT          REFERENCES pay_groups(id),   -- v2.0: WEEKLY_PUSAT / MONTHLY_OUTLET / ...
    employee_number     VARCHAR(50)     NOT NULL,
    full_name           VARCHAR(255)    NOT NULL,
    email               VARCHAR(255),                -- v2.0: opsional (banyak crew tidak punya email kerja); login via employee_number
    phone               VARCHAR(20),
    nik                 VARCHAR(20),               -- Nomor Induk Kependudukan
    npwp                VARCHAR(30),               -- Nomor Pokok Wajib Pajak
    gender              VARCHAR(10),               -- 'male', 'female'
    birth_date          DATE,
    join_date           DATE            NOT NULL,
    end_date            DATE,
    employment_status   VARCHAR(30)     NOT NULL DEFAULT 'permanent',
      -- values: 'permanent', 'contract', 'probation', 'intern', 'freelance', 'part_time'
    marital_status      VARCHAR(20)     DEFAULT 'single',
      -- values: 'single', 'married', 'divorced', 'widowed'
    dependent_count     SMALLINT        NOT NULL DEFAULT 0,  -- tanggungan PTKP
    bank_name           VARCHAR(100),
    bank_account_number VARCHAR(50),
    bank_account_name   VARCHAR(255),
    base_salary         NUMERIC(15,2)   NOT NULL DEFAULT 0,  -- total gaji/bulan (monthly_split); rincian di employee_salary_components
    daily_rate          NUMERIC(12,2),             -- v2.0: gaji harian (weekly_hourly), mis. 130.000
    hourly_rate         NUMERIC(12,2),             -- weekly_hourly: = daily_rate ÷ 9 (dihitung); freelance: tarif langsung (9.000)
    overtime_rate_multiplier NUMERIC(4,2) NOT NULL DEFAULT 1.5,  -- legacy; ABPS memakai pay_rules.overtime_flat_rate
    probation_until     DATE,                      -- v2.0: masa percobaan (mis. 3 bulan, 1,8jt → 2jt)
    ptkp_status         VARCHAR(10)     NOT NULL DEFAULT 'TK0',
      -- values: TK0, TK1, TK2, TK3, K0, K1, K2, K3, HB0 dst
    role                VARCHAR(30)     NOT NULL DEFAULT 'employee',
      -- values: 'super_admin','tenant_admin','hr_admin','hr_staff','manager','supervisor','employee'
    password_hash       TEXT            NOT NULL,
    avatar_url          TEXT,
    is_active           BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT uq_employee_number UNIQUE (tenant_id, employee_number),
    CONSTRAINT uq_employee_email  UNIQUE (tenant_id, email)   -- NULL diperbolehkan berulang
);

-- High-traffic indexes
CREATE INDEX idx_emp_tenant         ON employees(tenant_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_emp_position       ON employees(position_id);
CREATE INDEX idx_emp_location       ON employees(location_id);
CREATE INDEX idx_emp_role           ON employees(tenant_id, role);
CREATE INDEX idx_emp_status         ON employees(tenant_id, employment_status) WHERE deleted_at IS NULL;
CREATE INDEX idx_emp_fullname_trgm  ON employees USING GIN(full_name gin_trgm_ops);

-- ============================================================
-- 8. SHIFT TEMPLATES
-- ============================================================
CREATE TABLE shift_templates (
    id                      BIGSERIAL PRIMARY KEY,
    tenant_id               BIGINT          NOT NULL REFERENCES tenants(id),
    name                    VARCHAR(100)    NOT NULL,  -- "Pagi (P1)", "Siang (P2)", "Middle", "Shift 3"
    code                    VARCHAR(20),               -- v2.0: 'P1','P2','MID','S3','PRD'
    shift_type              VARCHAR(20)     NOT NULL DEFAULT 'fixed',
      -- values: 'fixed', 'rotating', 'flexible', 'split'
    start_time              TIME            NOT NULL,
    end_time                TIME            NOT NULL,
    is_overnight            BOOLEAN         NOT NULL DEFAULT FALSE,
    break_duration_minutes  INT             NOT NULL DEFAULT 60,
    tolerance_late_minutes  INT             NOT NULL DEFAULT 15,
    work_days               VARCHAR(20)     NOT NULL DEFAULT '1,2,3,4,5',
      -- comma-separated: 1=Mon, 7=Sun
    color_hex               VARCHAR(7)      DEFAULT '#3B82F6',
    is_active               BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at              TIMESTAMPTZ
);

CREATE INDEX idx_shifts_tenant ON shift_templates(tenant_id) WHERE deleted_at IS NULL;

-- ============================================================
-- 9. SHIFT ASSIGNMENTS (Jadwal karyawan per periode)
-- ============================================================
CREATE TABLE shift_assignments (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id         BIGINT          NOT NULL REFERENCES employees(id),
    shift_template_id   BIGINT          NOT NULL REFERENCES shift_templates(id),
    location_id         BIGINT          REFERENCES locations(id),
    effective_date      DATE            NOT NULL,
    end_date            DATE,
    recurrence          VARCHAR(20)     NOT NULL DEFAULT 'weekly',
      -- values: 'once', 'daily', 'weekly', 'biweekly', 'monthly'
    station_code        VARCHAR(10),               -- v2.0: 'ck','kit','ks','se','bk' (rotasi station per shift)
    day_status          VARCHAR(20)     NOT NULL DEFAULT 'work',
      -- v2.0 values: 'work','off','off_izin','off_sakit'
    assigned_by         BIGINT          NOT NULL REFERENCES employees(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sa_employee     ON shift_assignments(employee_id);
CREATE INDEX idx_sa_shift        ON shift_assignments(shift_template_id);
CREATE INDEX idx_sa_effective    ON shift_assignments(employee_id, effective_date, end_date);
CREATE INDEX idx_sa_tenant       ON shift_assignments(tenant_id);

-- ============================================================
-- 10. SHIFT SWAP REQUESTS
-- ============================================================
CREATE TABLE shift_swap_requests (
    id                      BIGSERIAL PRIMARY KEY,
    tenant_id               BIGINT          NOT NULL REFERENCES tenants(id),
    requester_id            BIGINT          NOT NULL REFERENCES employees(id),
    target_employee_id      BIGINT          NOT NULL REFERENCES employees(id),
    requester_assignment_id BIGINT          NOT NULL REFERENCES shift_assignments(id),
    target_assignment_id    BIGINT          NOT NULL REFERENCES shift_assignments(id),
    swap_date               DATE            NOT NULL,
    reason                  TEXT,
    status                  VARCHAR(20)     NOT NULL DEFAULT 'pending',
      -- values: 'pending', 'target_approved', 'supervisor_approved', 'rejected', 'cancelled'
    target_approved_at      TIMESTAMPTZ,
    supervisor_id           BIGINT          REFERENCES employees(id),
    supervisor_approved_at  TIMESTAMPTZ,
    reject_reason           TEXT,
    created_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_swap_requester ON shift_swap_requests(requester_id);
CREATE INDEX idx_swap_target    ON shift_swap_requests(target_employee_id);
CREATE INDEX idx_swap_status    ON shift_swap_requests(tenant_id, status);

-- ============================================================
-- 11. ATTENDANCES (Core — high write volume)
-- ============================================================
CREATE TABLE attendances (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id         BIGINT          NOT NULL REFERENCES employees(id),
    location_id         BIGINT          REFERENCES locations(id),
    shift_assignment_id BIGINT          REFERENCES shift_assignments(id),
    attendance_date     DATE            NOT NULL,
    clock_in_at         TIMESTAMPTZ,
    clock_out_at        TIMESTAMPTZ,
    clock_in_lat        NUMERIC(10,7),
    clock_in_lng        NUMERIC(10,7),
    clock_in_accuracy   NUMERIC(8,2),    -- meters
    clock_out_lat       NUMERIC(10,7),
    clock_out_lng       NUMERIC(10,7),
    clock_out_accuracy  NUMERIC(8,2),
    clock_in_distance   NUMERIC(10,2),   -- jarak dari pusat lokasi (meter)
    clock_in_photo_url  TEXT,
    clock_out_photo_url TEXT,
    status              VARCHAR(30)     NOT NULL DEFAULT 'present',
      -- values: 'present','absent','late','early_leave','leave','sick','holiday','off'
    clock_in_status     VARCHAR(20)     NOT NULL DEFAULT 'valid',
      -- values: 'valid', 'outside_radius', 'mock_gps', 'manual', 'system'
    is_mock_gps         BOOLEAN         NOT NULL DEFAULT FALSE,
    fraud_flag          BOOLEAN         NOT NULL DEFAULT FALSE,
    late_minutes        INT             NOT NULL DEFAULT 0,
    early_leave_minutes INT             NOT NULL DEFAULT 0,
    effective_work_minutes INT          NOT NULL DEFAULT 0,
    overtime_minutes    INT             NOT NULL DEFAULT 0,
    outside_radius_note TEXT,
    approved_by         BIGINT          REFERENCES employees(id),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_attendance_emp_date UNIQUE (employee_id, attendance_date)
);

-- High-traffic indexes (query dominan: per employee per range, per tenant per date)
CREATE INDEX idx_att_employee_date  ON attendances(employee_id, attendance_date DESC);
CREATE INDEX idx_att_tenant_date    ON attendances(tenant_id, attendance_date DESC);
CREATE INDEX idx_att_location       ON attendances(location_id);
CREATE INDEX idx_att_status         ON attendances(tenant_id, status, attendance_date);
CREATE INDEX idx_att_fraud          ON attendances(tenant_id, fraud_flag) WHERE fraud_flag = TRUE;
CREATE INDEX idx_att_late           ON attendances(employee_id, attendance_date) WHERE late_minutes > 0;

-- ============================================================
-- 12. LEAVE REQUESTS (Izin & Cuti)
-- ============================================================
CREATE TABLE leave_requests (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id     BIGINT          NOT NULL REFERENCES employees(id),
    leave_type      VARCHAR(30)     NOT NULL,
      -- values: 'annual','sick','unpaid','maternity','paternity','emergency','other'
    start_date      DATE            NOT NULL,
    end_date        DATE            NOT NULL,
    total_days      SMALLINT        NOT NULL,
    is_paid         BOOLEAN         NOT NULL DEFAULT TRUE,
    reason          TEXT,
    document_url    TEXT,
    status          VARCHAR(20)     NOT NULL DEFAULT 'pending',
      -- values: 'pending', 'approved', 'rejected', 'cancelled'
    reviewed_by     BIGINT          REFERENCES employees(id),
    reviewed_at     TIMESTAMPTZ,
    reject_reason   TEXT,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_leave_employee   ON leave_requests(employee_id, start_date DESC);
CREATE INDEX idx_leave_tenant     ON leave_requests(tenant_id, status);
CREATE INDEX idx_leave_date_range ON leave_requests(employee_id, start_date, end_date);

-- ============================================================
-- 13. OVERTIME REQUESTS
-- ============================================================
CREATE TABLE overtime_requests (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id     BIGINT          NOT NULL REFERENCES employees(id),
    attendance_id   BIGINT          REFERENCES attendances(id),
    overtime_date   DATE            NOT NULL,
    start_time      TIMESTAMPTZ     NOT NULL,
    end_time        TIMESTAMPTZ     NOT NULL,
    duration_minutes INT            NOT NULL,
    rate_mode       VARCHAR(20)     NOT NULL DEFAULT 'flat_per_hour', -- v2.0: 'flat_per_hour' | 'multiplier'
    flat_rate       NUMERIC(15,2),                  -- snapshot pay_rules.overtime_flat_rate (10.000 / 17.000)
    payable_hours   NUMERIC(5,2),                   -- setelah aturan 9 jam + grace 30 mnt + roundup (pusat) / input jam (bulanan)
    multiplier      NUMERIC(4,2)    NOT NULL DEFAULT 1.0,  -- legacy, tidak dipakai ABPS
    reason          TEXT,
    status          VARCHAR(20)     NOT NULL DEFAULT 'pending',
      -- values: 'pending', 'approved', 'rejected'
    approved_by     BIGINT          REFERENCES employees(id),
    approved_at     TIMESTAMPTZ,
    reject_reason   TEXT,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ot_employee ON overtime_requests(employee_id, overtime_date DESC);
CREATE INDEX idx_ot_tenant   ON overtime_requests(tenant_id, status);

-- ============================================================
-- 14. SALARY COMPONENTS (Master Komponen Gaji)
-- ============================================================
CREATE TABLE salary_components (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT          NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255)    NOT NULL,  -- "Tunjangan Makan", "BPJS Kesehatan"
    code            VARCHAR(50)     NOT NULL,  -- "TUNJ_MAKAN", "BPJS_KES"
    component_type  VARCHAR(20)     NOT NULL,
      -- values: 'earning_fixed','earning_variable','deduction_fixed','deduction_variable','tax','bpjs'
    is_taxable      BOOLEAN         NOT NULL DEFAULT TRUE,
    is_bpjs_base    BOOLEAN         NOT NULL DEFAULT FALSE,
    default_amount  NUMERIC(15,2)   NOT NULL DEFAULT 0,
    default_rate    NUMERIC(8,4)    NOT NULL DEFAULT 0,
    calculation_basis VARCHAR(20)   NOT NULL DEFAULT 'fixed',
      -- values: 'fixed', 'percent_of_base', 'percent_of_total_earn'
    is_system       BOOLEAN         NOT NULL DEFAULT FALSE,  -- komponen sistem (BPJS, PPh21) tidak bisa hapus
    is_active       BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_component_code UNIQUE (tenant_id, code)
);

CREATE INDEX idx_comp_tenant ON salary_components(tenant_id);

-- ============================================================
-- 15. EMPLOYEE SALARY COMPONENTS (Komponen per karyawan)
-- ============================================================
CREATE TABLE employee_salary_components (
    id                      BIGSERIAL PRIMARY KEY,
    tenant_id               BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id             BIGINT          NOT NULL REFERENCES employees(id),
    salary_component_id     BIGINT          NOT NULL REFERENCES salary_components(id),
    amount                  NUMERIC(15,2)   NOT NULL DEFAULT 0,
    rate                    NUMERIC(8,4)    NOT NULL DEFAULT 0,
    effective_from          DATE            NOT NULL,
    effective_until         DATE,
    notes                   TEXT,
    created_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_esc_employee  ON employee_salary_components(employee_id);
CREATE INDEX idx_esc_component ON employee_salary_components(salary_component_id);
CREATE INDEX idx_esc_effective ON employee_salary_components(employee_id, effective_from, effective_until);

-- ============================================================
-- 16. PAYROLL RUNS (Periode Penggajian)
-- ============================================================
CREATE TABLE payroll_runs (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    pay_group_id        BIGINT          NOT NULL REFERENCES pay_groups(id),  -- v2.0
    pay_frequency       VARCHAR(20)     NOT NULL,  -- 'weekly' | 'monthly' (snapshot)
    period_start        DATE            NOT NULL,  -- weekly: Sabtu; monthly: tgl 1
    period_end          DATE            NOT NULL,  -- weekly: Jumat; monthly: akhir bulan
    period_year         SMALLINT        NOT NULL,  -- turunan period_end (untuk laporan)
    period_month        SMALLINT        NOT NULL,
    period_label        VARCHAR(50)     NOT NULL,  -- "Oktober 2026" / "Minggu 10–16 Okt 2026"
    pay_date            DATE,
    status              VARCHAR(20)     NOT NULL DEFAULT 'draft',
      -- values: 'draft', 'calculated', 'reviewed', 'approved', 'paid', 'cancelled'  (selaras prototipe)
    total_employees     INT             NOT NULL DEFAULT 0,
    total_gross         NUMERIC(18,2)   NOT NULL DEFAULT 0,
    total_deductions    NUMERIC(18,2)   NOT NULL DEFAULT 0,
    total_net           NUMERIC(18,2)   NOT NULL DEFAULT 0,
    generated_by        BIGINT          NOT NULL REFERENCES employees(id),
    generated_at        TIMESTAMPTZ,
    reviewed_by         BIGINT          REFERENCES employees(id),
    reviewed_at         TIMESTAMPTZ,
    approved_by         BIGINT          REFERENCES employees(id),
    approved_at         TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_payroll_run UNIQUE (tenant_id, pay_group_id, period_start),
    CONSTRAINT ck_period CHECK (period_end >= period_start)
);

CREATE INDEX idx_pr_tenant  ON payroll_runs(tenant_id, period_end DESC);
CREATE INDEX idx_pr_status  ON payroll_runs(tenant_id, status);

-- ============================================================
-- 17. PAYROLL ITEMS (Per Karyawan per Run)
-- ============================================================
CREATE TABLE payroll_items (
    id                      BIGSERIAL PRIMARY KEY,
    tenant_id               BIGINT          NOT NULL REFERENCES tenants(id),
    payroll_run_id          BIGINT          NOT NULL REFERENCES payroll_runs(id),
    employee_id             BIGINT          NOT NULL REFERENCES employees(id),
    payroll_type            VARCHAR(20)     NOT NULL,  -- snapshot positions.payroll_type
    -- Rekap kehadiran snapshot
    working_days            SMALLINT        NOT NULL DEFAULT 0,
    present_days            SMALLINT        NOT NULL DEFAULT 0,
    absent_days             SMALLINT        NOT NULL DEFAULT 0,
    late_count              SMALLINT        NOT NULL DEFAULT 0,
    late_total_minutes      INT             NOT NULL DEFAULT 0,
    total_work_hours        NUMERIC(8,2)    NOT NULL DEFAULT 0,
    overtime_hours          NUMERIC(8,2)    NOT NULL DEFAULT 0,
    izin_days               NUMERIC(4,1)    NOT NULL DEFAULT 0,  -- v2.0
    sick_days               NUMERIC(4,1)    NOT NULL DEFAULT 0,
    full_9h_days            SMALLINT        NOT NULL DEFAULT 0,  -- pusat: hari dibayar flat gaji harian
    early_arrival_count     SMALLINT        NOT NULL DEFAULT 0,  -- pusat: masuk < 05:00
    holiday_worked_count    SMALLINT        NOT NULL DEFAULT 0,
    -- Kalkulasi v2.0 (ringkasan; rincian di payroll_item_details)
    total_kasbon            NUMERIC(15,2)   NOT NULL DEFAULT 0,
    total_bonus             NUMERIC(15,2)   NOT NULL DEFAULT 0,
    total_holdback          NUMERIC(15,2)   NOT NULL DEFAULT 0,
    transfer_bank           VARCHAR(50),                         -- snapshot (mis. BCA), tanpa nomor rekening
    -- Kalkulasi
    base_salary             NUMERIC(15,2)   NOT NULL DEFAULT 0,
    total_earnings          NUMERIC(15,2)   NOT NULL DEFAULT 0,
    total_deductions        NUMERIC(15,2)   NOT NULL DEFAULT 0,
    total_bpjs_employee     NUMERIC(15,2)   NOT NULL DEFAULT 0,  -- ABPS: 0 selama BPJS non-aktif
    total_bpjs_company      NUMERIC(15,2)   NOT NULL DEFAULT 0,
    pph21                   NUMERIC(15,2)   NOT NULL DEFAULT 0,
    net_salary              NUMERIC(15,2)   NOT NULL DEFAULT 0,  -- = THP (earnings − deductions − kasbon − holdback)
    slip_url                TEXT,
    slip_generated_at       TIMESTAMPTZ,
    status                  VARCHAR(20)     NOT NULL DEFAULT 'calculated',
      -- values: 'calculated', 'approved', 'paid'
    created_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_payroll_item UNIQUE (payroll_run_id, employee_id)
);

-- High-traffic: lookup slip gaji per karyawan
CREATE INDEX idx_pi_employee    ON payroll_items(employee_id, payroll_run_id DESC);
CREATE INDEX idx_pi_run         ON payroll_items(payroll_run_id);
CREATE INDEX idx_pi_tenant      ON payroll_items(tenant_id);
CREATE INDEX idx_pi_status      ON payroll_items(payroll_run_id, status);

-- ============================================================
-- 18. PAYROLL ITEM DETAILS (Breakdown komponen per item)
-- ============================================================
CREATE TABLE payroll_item_details (
    id                      BIGSERIAL PRIMARY KEY,
    payroll_item_id         BIGINT          NOT NULL REFERENCES payroll_items(id),
    salary_component_id     BIGINT          NOT NULL REFERENCES salary_components(id),
    quantity                NUMERIC(10,2),             -- v2.0: jam / menit / hari / kg / task
    unit_rate               NUMERIC(15,2),             -- v2.0: tarif per unit (snapshot)
    component_name          VARCHAR(255)    NOT NULL,  -- snapshot nama saat itu
    component_type          VARCHAR(20)     NOT NULL,
    amount                  NUMERIC(15,2)   NOT NULL DEFAULT 0,
    notes                   TEXT,
    created_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pid_item       ON payroll_item_details(payroll_item_id);
CREATE INDEX idx_pid_component  ON payroll_item_details(salary_component_id);

-- ============================================================
-- 19. AUDIT LOGS
-- ============================================================
CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT          NOT NULL REFERENCES tenants(id),
    actor_id        BIGINT          REFERENCES employees(id),
    action          VARCHAR(100)    NOT NULL,  -- 'attendance.clock_in', 'payroll.approved', dll
    entity_type     VARCHAR(50)     NOT NULL,
    entity_id       BIGINT,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_al_tenant      ON audit_logs(tenant_id, created_at DESC);
CREATE INDEX idx_al_actor       ON audit_logs(actor_id, created_at DESC);
CREATE INDEX idx_al_entity      ON audit_logs(entity_type, entity_id);

-- ============================================================
-- 20. NOTIFICATIONS
-- ============================================================
CREATE TABLE notifications (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id     BIGINT          NOT NULL REFERENCES employees(id),
    type            VARCHAR(50)     NOT NULL,  -- 'clock_in_reminder', 'payslip_ready', dll
    title           VARCHAR(255)    NOT NULL,
    body            TEXT,
    is_read         BOOLEAN         NOT NULL DEFAULT FALSE,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notif_employee ON notifications(employee_id, is_read, created_at DESC);
CREATE INDEX idx_notif_unread   ON notifications(employee_id) WHERE is_read = FALSE;
```

---

---

## Tabel Baru v2.0 (Kebutuhan Lapangan ABPS)

```sql
-- ============================================================
-- 21. PAY GROUPS (Kelompok penggajian: frekuensi & periode)
-- ============================================================
CREATE TABLE pay_groups (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    code                VARCHAR(50)     NOT NULL,   -- 'WEEKLY_PUSAT','MONTHLY_OUTLET','MONTHLY_KANTOR','PART_TIME'
    name                VARCHAR(255)    NOT NULL,
    pay_frequency       VARCHAR(20)     NOT NULL,
      -- values: 'weekly', 'monthly'
    week_start_dow      SMALLINT,                    -- 6 = Sabtu (periode Sabtu–Jumat) untuk weekly
    month_cutoff_day    SMALLINT,                    -- NULL = kalender (1..akhir bulan). Lihat Q-07
    payday_rule         VARCHAR(100),                -- teks/konfigurasi, mis. 'weekly:friday', 'monthly:3' (jadwal crew menandai "3/payday")
    is_active           BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_pay_group UNIQUE (tenant_id, code)
);

-- ============================================================
-- 22. PAY RULES (parameter kalkulasi, versi per tanggal berlaku)
--     SEMUA angka aturan payroll dibaca dari sini, TIDAK di-hardcode.
-- ============================================================
-- v2.1: pay_rules dipertahankan sebagai VIEW/kompatibilitas; sumber kebenaran = setting_values (bagian v2.1)
CREATE TABLE pay_rules (
    id                          BIGSERIAL PRIMARY KEY,
    tenant_id                   BIGINT          NOT NULL REFERENCES tenants(id),
    pay_group_id                BIGINT          NOT NULL REFERENCES pay_groups(id),
    effective_from              DATE            NOT NULL,
    effective_until             DATE,
    -- Basis hari/jam
    divisor_days_per_month      SMALLINT        NOT NULL DEFAULT 26,     -- gaji/hari = gaji bulan ÷ 26
    standard_hours_per_day      NUMERIC(4,2)    NOT NULL DEFAULT 9,      -- gaji/jam = gaji harian ÷ 9 (pusat)
    -- Clamp jam masuk (pusat): jam hitung = MAX(jam masuk, earliest_count_time)
    earliest_count_time         TIME,                                     -- '05:00' (pusat)
    early_arrival_incentive     NUMERIC(15,2)   NOT NULL DEFAULT 0,       -- Rp10.000 jika masuk < 05:00
    early_arrival_before        TIME,                                     -- '05:00'
    -- Lembur
    overtime_mode               VARCHAR(20)     NOT NULL DEFAULT 'flat_per_hour',
      -- values: 'flat_per_hour', 'multiplier'
    overtime_flat_rate          NUMERIC(15,2)   NOT NULL DEFAULT 10000,  -- crew & pusat Rp10.000/jam; kantor 17.000 (lihat Q-16)
    overtime_threshold_minutes  INT             NOT NULL DEFAULT 540,    -- > 9 jam kerja
    overtime_grace_minutes      INT             NOT NULL DEFAULT 30,     -- kelebihan ≤ 30 mnt tidak dihitung
    overtime_rounding           VARCHAR(20)     NOT NULL DEFAULT 'ceil_hour',
      -- 'ceil_hour' = ROUNDUP((durasi − 9j − 30m) jam, 0); bulanan memakai input jam (boleh 0,5)
    -- Keterlambatan (bulanan)
    late_deduction_per_minute   NUMERIC(15,2)   NOT NULL DEFAULT 500,    -- crew outlet Rp500/menit
    late_deduction_mode         VARCHAR(30)     NOT NULL DEFAULT 'per_minute',
      -- 'per_minute' (crew) | 'prorata_hourly' (kantor: menit × gaji/26/9 ÷ 60)
    -- Izin / sakit / absen (bulanan)
    izin_deduction_basis        VARCHAR(30)     NOT NULL DEFAULT 'total_salary_div_days',
      -- crew: hari × (gaji total ÷ 26); kantor: hari × (tunjangan tidak tetap ÷ 26)
    sakit_deduction_basis       VARCHAR(30)     NOT NULL DEFAULT 'variable_allowance_div_days',
      -- crew: hari × (tunj. tidak tetap ÷ 26); kantor (Sept 2026): Rp50.000/hari
    sakit_flat_per_day          NUMERIC(15,2),
    izin_flat_per_day           NUMERIC(15,2),                            -- Maret 2026 sempat Rp80.000/hari
    -- Hari Minggu / tanggal merah (pusat)
    sunday_policy               VARCHAR(30)     NOT NULL DEFAULT 'hourly_normal',  -- lihat Q-03
    notes                       TEXT,
    created_at                  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_pay_rules_group ON pay_rules(pay_group_id, effective_from DESC);

-- ============================================================
-- 23. HOLIDAYS (Tanggal merah + insentif)
-- ============================================================
CREATE TABLE holidays (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    holiday_date        DATE            NOT NULL,
    name                VARCHAR(255)    NOT NULL,   -- "Hari Lahir Pancasila"
    incentive_amount    NUMERIC(15,2)   NOT NULL DEFAULT 50000, -- data lapangan: Rp50.000 (pusat & outlet)
    applies_to_group_ids BIGINT[],                  -- NULL = semua pay_group
    is_closure          BOOLEAN         NOT NULL DEFAULT FALSE, -- libur operasional (mis. Lebaran 18–28 Mar 2026, dibayar 50%)
    closure_pay_percent NUMERIC(5,2),                -- 50.00 untuk periode libur Lebaran
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_holiday UNIQUE (tenant_id, holiday_date)
);

-- ============================================================
-- 24. CASH ADVANCES (Kasbon) & cicilan potongan
-- ============================================================
CREATE TABLE cash_advances (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id         BIGINT          NOT NULL REFERENCES employees(id),
    advance_date        DATE            NOT NULL,
    amount              NUMERIC(15,2)   NOT NULL,
    source              VARCHAR(50),                -- 'kas_outlet','kas_besar','abah' (sheet Agustus: "Kasbon Abah")
    installment_amount  NUMERIC(15,2),              -- NULL = potong penuh di periode berikutnya
    remaining_amount    NUMERIC(15,2)   NOT NULL,
    status              VARCHAR(20)     NOT NULL DEFAULT 'open',  -- 'open','settled','cancelled'
    approved_by         BIGINT          REFERENCES employees(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_ca_emp ON cash_advances(employee_id, status);

CREATE TABLE cash_advance_repayments (
    id                  BIGSERIAL PRIMARY KEY,
    cash_advance_id     BIGINT          NOT NULL REFERENCES cash_advances(id),
    payroll_item_id     BIGINT          REFERENCES payroll_items(id),
    amount              NUMERIC(15,2)   NOT NULL,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 25. PIECE RATE (Borongan: kurir per tugas, metik cengek/baput per kg)
-- ============================================================
CREATE TABLE piece_rate_items (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    code                VARCHAR(50)     NOT NULL,  -- 'KURIR_ANTAR_BAHAN','KURIR_SAMPAH_TPS','KURIR_ANTAR_PESANAN','METIK_CENGEK','METIK_BAPUT'
    name                VARCHAR(255)    NOT NULL,
    unit                VARCHAR(20)     NOT NULL,  -- 'task','kg'
    rate                NUMERIC(15,2)   NOT NULL,  -- 20.000/task; 2.500/kg cengek; 3.000/kg baput
    effective_from      DATE            NOT NULL,
    is_active           BOOLEAN         NOT NULL DEFAULT TRUE,
    CONSTRAINT uq_piece_item UNIQUE (tenant_id, code, effective_from)
);

CREATE TABLE piece_work_entries (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id         BIGINT          NOT NULL REFERENCES employees(id),
    piece_rate_item_id  BIGINT          NOT NULL REFERENCES piece_rate_items(id),
    work_date           DATE            NOT NULL,
    quantity            NUMERIC(10,2)   NOT NULL,
    rate_snapshot       NUMERIC(15,2)   NOT NULL,
    amount              NUMERIC(15,2)   NOT NULL,   -- quantity × rate_snapshot
    recorded_by         BIGINT          REFERENCES employees(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_pwe_emp_date ON piece_work_entries(employee_id, work_date);

-- ============================================================
-- 26. VISIT LOGS (Part-time per kunjungan, mis. Content Specialist Rp50.000/visit)
-- ============================================================
CREATE TABLE visit_logs (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id         BIGINT          NOT NULL REFERENCES employees(id),
    visit_date          DATE            NOT NULL,
    attendance_id       BIGINT          REFERENCES attendances(id),
    rate_snapshot       NUMERIC(15,2)   NOT NULL DEFAULT 50000,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_visit UNIQUE (employee_id, visit_date)
);

-- ============================================================
-- 27. OUTLET REVENUES (omset bulanan per outlet untuk bonus)
-- ============================================================
CREATE TABLE outlet_revenues (
    id                      BIGSERIAL PRIMARY KEY,
    tenant_id               BIGINT          NOT NULL REFERENCES tenants(id),
    location_id             BIGINT          NOT NULL REFERENCES locations(id),
    period_year             SMALLINT        NOT NULL,
    period_month            SMALLINT        NOT NULL,
    gross_revenue           NUMERIC(18,2)   NOT NULL,
    discount_amount         NUMERIC(18,2)   NOT NULL DEFAULT 0,
    ojol_commission_amount  NUMERIC(18,2)   NOT NULL DEFAULT 0,   -- potongan aplikasi GoFood/GrabFood/ShopeeFood
    net_revenue             NUMERIC(18,2)   GENERATED ALWAYS AS (gross_revenue - discount_amount - ojol_commission_amount) STORED,
    source                  VARCHAR(50),                          -- 'manual','moka','runchise','kledo' (sumber yang terlihat di Kas Outlet)
    headcount_snapshot      SMALLINT,                              -- aturan bonus ditulis "kondisi 12 karyawan"
    input_by                BIGINT          REFERENCES employees(id),
    verified_by             BIGINT          REFERENCES employees(id),
    created_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_outlet_rev UNIQUE (tenant_id, location_id, period_year, period_month)
);

-- ============================================================
-- 28. REVENUE BONUS TIERS (aturan bonus outlet)
-- ============================================================
CREATE TABLE revenue_bonus_tiers (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    location_id         BIGINT          REFERENCES locations(id),   -- NULL = semua outlet
    bonus_type          VARCHAR(30)     NOT NULL DEFAULT 'outlet',  -- 'outlet','catering'
    min_net_revenue     NUMERIC(18,2)   NOT NULL,                  -- inklusif
    max_net_revenue     NUMERIC(18,2),                             -- NULL = tak terbatas
    percent_of_net      NUMERIC(6,3)    NOT NULL,                  -- 0 / 0.5 / 1.0
    reference_headcount SMALLINT,                                  -- 12
    effective_from      DATE            NOT NULL,
    effective_until     DATE
);
-- Seed: [0, 350jt) → 0% ; [350jt, 400jt] → 0,5% ; (400jt, ∞) → 1%  (batas tepat 400jt: lihat Q-01)

-- ============================================================
-- 29. BONUS POOLS & ALLOCATIONS (pembagian bonus ke karyawan)
-- ============================================================
CREATE TABLE bonus_pools (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    outlet_revenue_id   BIGINT          REFERENCES outlet_revenues(id),
    bonus_type          VARCHAR(30)     NOT NULL,     -- 'outlet','catering','pelayanan','other'
    period_year         SMALLINT        NOT NULL,
    period_month        SMALLINT        NOT NULL,
    tier_percent        NUMERIC(6,3),
    pool_amount         NUMERIC(15,2)   NOT NULL,     -- net_revenue × tier_percent
    allocation_method   VARCHAR(30)     NOT NULL DEFAULT 'manual',
      -- 'manual' | 'equal' | 'attendance_weighted'  (lapangan Sept 2026: 180rb vs 80rb, lihat Q-02)
    status              VARCHAR(20)     NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE TABLE bonus_allocations (
    id                  BIGSERIAL PRIMARY KEY,
    bonus_pool_id       BIGINT          NOT NULL REFERENCES bonus_pools(id),
    employee_id         BIGINT          NOT NULL REFERENCES employees(id),
    amount              NUMERIC(15,2)   NOT NULL,
    reason              TEXT,                         -- mis. "ada izin 1 hari"
    CONSTRAINT uq_bonus_alloc UNIQUE (bonus_pool_id, employee_id)
);

-- ============================================================
-- 30. SALARY HOLDBACKS (Gaji gantungan — sheet MARET 2026)
-- ============================================================
CREATE TABLE salary_holdbacks (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    employee_id         BIGINT          NOT NULL REFERENCES employees(id),
    held_in_item_id     BIGINT          REFERENCES payroll_items(id),
    released_in_item_id BIGINT          REFERENCES payroll_items(id),
    amount              NUMERIC(15,2)   NOT NULL,     -- lapangan: 450.000 per minggu-1 & minggu-2 Maret
    reason              TEXT,
    status              VARCHAR(20)     NOT NULL DEFAULT 'held',  -- 'held','released'
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
```

---

---

## v2.1 — Registry Pengaturan Dinamis (Rule Settings)

> Mulai v2.1 **semua** angka, toggle, dan pilihan metode payroll/presensi disimpan di registry ini. Kolom aturan di `pay_rules` (v2.0) menjadi **view kompatibilitas** yang dibaca dari registry. Tabel domain (`revenue_bonus_tiers`, `piece_rate_items`, `holidays`) tetap ada untuk data berbentuk daftar, dan juga berversi tanggal serta diaudit.

### Urutan override (paling spesifik menang)
`employee` → `position` → `location` → `pay_group` → `tenant` → `definition.default_value`
Pada scope yang sama, versi yang berlaku = `effective_from <= tanggal_acuan < COALESCE(effective_to, ∞)` dengan `status = 'active'`. Tanggal acuan = `period_end` payroll run (atau tanggal presensi untuk aturan harian).

```sql
-- ============================================================
-- 31. SETTING DEFINITIONS (katalog key — dikelola developer via migration/seed)
-- ============================================================
CREATE TABLE setting_definitions (
    id                  BIGSERIAL PRIMARY KEY,
    key                 VARCHAR(150)    NOT NULL UNIQUE,   -- 'overtime.grace_minutes', 'feature.bpjs'
    category            VARCHAR(50)     NOT NULL,          -- 'feature','overtime','attendance','deduction','bonus','period','weekly','revenue','slip','rbac','system'
    label               VARCHAR(255)    NOT NULL,
    description         TEXT,
    value_type          VARCHAR(20)     NOT NULL,
      -- 'boolean','integer','decimal','money','percent','time','date','enum','multi_enum','text','json'
    json_schema         JSONB,                             -- validasi untuk value_type json/enum (zod di-generate dari sini)
    default_value       JSONB           NOT NULL,
    min_value           NUMERIC,
    max_value           NUMERIC,
    allowed_scopes      VARCHAR(20)[]   NOT NULL DEFAULT '{tenant}',
      -- subset {tenant,pay_group,location,position,employee}
    depends_on_key      VARCHAR(150),                      -- mis. 'feature.bpjs' untuk 'bpjs.kes.employee_percent'
    edit_permission     VARCHAR(50)     NOT NULL DEFAULT 'settings.edit',  -- permission RBAC
    requires_approval   BOOLEAN         NOT NULL DEFAULT FALSE,            -- v2.2: HANYA default seed; sumber kebenaran = approval_policies
    is_locked           BOOLEAN         NOT NULL DEFAULT FALSE,            -- tidak bisa diubah via UI (mis. system.public_api)
    question_ref        VARCHAR(10),                       -- 'Q-04', 'A3' (keterlacakan ke CHANGELOG)
    sort_order          INT             NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

-- ============================================================
-- 32. SETTING OPTIONS (pilihan untuk enum/multi_enum)
-- ============================================================
CREATE TABLE setting_options (
    id                  BIGSERIAL PRIMARY KEY,
    definition_id       BIGINT          NOT NULL REFERENCES setting_definitions(id),
    value               VARCHAR(100)    NOT NULL,          -- 'equal','attendance_days','position_weight','manual'
    label               VARCHAR(255)    NOT NULL,          -- 'Rata', 'Proporsional hari hadir'
    description         TEXT,
    extra_params_schema JSONB,                             -- parameter tambahan saat opsi ini dipilih
    is_active           BOOLEAN         NOT NULL DEFAULT TRUE,
    sort_order          INT             NOT NULL DEFAULT 0,
    CONSTRAINT uq_setting_option UNIQUE (definition_id, value)
);

-- ============================================================
-- 33. SETTING VALUES (nilai per scope, berversi tanggal)
-- ============================================================
CREATE TABLE setting_values (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    definition_id       BIGINT          NOT NULL REFERENCES setting_definitions(id),
    scope_type          VARCHAR(20)     NOT NULL,          -- 'tenant','pay_group','location','position','employee'
    scope_id            BIGINT,                            -- NULL jika scope_type = 'tenant'
    value               JSONB           NOT NULL,          -- {"v": 30} / {"v": "ceil_hour"} / {"v": true} / {"v": [...]} 
    effective_from      DATE            NOT NULL,
    effective_to        DATE,                              -- eksklusif; NULL = tanpa batas
    status              VARCHAR(20)     NOT NULL DEFAULT 'active',
      -- 'draft','pending_approval','active','scheduled','superseded','rejected'
    reason              TEXT            NOT NULL,          -- alasan perubahan wajib
    created_by          BIGINT          NOT NULL REFERENCES employees(id),
    approved_by         BIGINT          REFERENCES employees(id),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT ck_scope CHECK ((scope_type = 'tenant') = (scope_id IS NULL)),
    CONSTRAINT ck_range CHECK (effective_to IS NULL OR effective_to > effective_from),
    -- tidak boleh ada dua versi active yang overlap untuk key+scope yang sama
    CONSTRAINT ex_no_overlap EXCLUDE USING gist (
        tenant_id WITH =, definition_id WITH =, scope_type WITH =, COALESCE(scope_id,0) WITH =,
        daterange(effective_from, effective_to, '[)') WITH &&
    ) WHERE (status = 'active')
);
-- butuh: CREATE EXTENSION btree_gist;
CREATE INDEX idx_sv_lookup ON setting_values(tenant_id, definition_id, scope_type, scope_id, effective_from DESC) WHERE status = 'active';

-- ============================================================
-- 34. SETTING AUDIT LOG (append-only)
-- ============================================================
CREATE TABLE setting_audit_logs (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    setting_value_id    BIGINT          REFERENCES setting_values(id),
    definition_key      VARCHAR(150)    NOT NULL,
    action              VARCHAR(20)     NOT NULL,          -- 'create','update','approve','reject','supersede','delete_draft'
    scope_type          VARCHAR(20)     NOT NULL,
    scope_id            BIGINT,
    old_value           JSONB,
    new_value           JSONB,
    old_effective       DATERANGE,
    new_effective       DATERANGE,
    reason              TEXT,
    actor_id            BIGINT          NOT NULL REFERENCES employees(id),
    actor_ip            INET,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
-- REVOKE UPDATE, DELETE ON setting_audit_logs FROM app_role;  (append-only)
CREATE INDEX idx_sal_key ON setting_audit_logs(tenant_id, definition_key, created_at DESC);

-- ============================================================
-- 35. PAYROLL RUN SETTING SNAPSHOT (beku saat run calculated/approved)
-- ============================================================
CREATE TABLE payroll_run_snapshots (
    id                  BIGSERIAL PRIMARY KEY,
    payroll_run_id      BIGINT          NOT NULL REFERENCES payroll_runs(id),
    employee_id         BIGINT          REFERENCES employees(id),   -- NULL = setting level run
    resolved_settings   JSONB           NOT NULL,  -- { "overtime.grace_minutes": {"v":30,"source":"pay_group:1","setting_value_id":88}, ... }
    domain_snapshot     JSONB           NOT NULL,  -- tier bonus, tarif borongan, holidays, komponen gaji yang dipakai
    settings_hash       CHAR(64)        NOT NULL,  -- sha256 untuk deteksi perubahan
    engine_version      VARCHAR(20)     NOT NULL,  -- versi kode calculator
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_run_snapshot UNIQUE (payroll_run_id, employee_id)
);
-- payroll_runs: tambah kolom
ALTER TABLE payroll_runs ADD COLUMN settings_frozen_at TIMESTAMPTZ;   -- diisi saat approve; setelah itu snapshot immutable
ALTER TABLE payroll_runs ADD COLUMN run_type VARCHAR(20) NOT NULL DEFAULT 'regular'; -- 'regular','adjustment'
ALTER TABLE payroll_runs ADD COLUMN adjusts_run_id BIGINT REFERENCES payroll_runs(id);

-- ============================================================
-- 36. TABEL PENDUKUNG OPSI
-- ============================================================
CREATE TABLE bonus_position_weights (       -- untuk opsi allocation_method = 'position_weight'
    id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    position_id BIGINT NOT NULL REFERENCES positions(id), weight NUMERIC(6,3) NOT NULL DEFAULT 1,
    effective_from DATE NOT NULL, effective_to DATE
);
CREATE TABLE revenue_imports (              -- untuk revenue.source = import_*
    id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    location_id BIGINT NOT NULL REFERENCES locations(id),
    source VARCHAR(30) NOT NULL,             -- 'manual','moka','runchise','kledo','gofood','grabfood','shopeefood'
    file_url TEXT, period_start DATE NOT NULL, period_end DATE NOT NULL,
    gross NUMERIC(18,2), discount NUMERIC(18,2), ojol_commission NUMERIC(18,2), other_deductions JSONB,
    status VARCHAR(20) NOT NULL DEFAULT 'uploaded',  -- 'uploaded','parsed','verified','rejected'
    uploaded_by BIGINT NOT NULL REFERENCES employees(id), verified_by BIGINT REFERENCES employees(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
ALTER TABLE outlet_revenues ADD COLUMN revenue_import_ids BIGINT[];  -- agregasi dari import
ALTER TABLE revenue_bonus_tiers ADD COLUMN min_inclusive BOOLEAN NOT NULL DEFAULT TRUE;
ALTER TABLE revenue_bonus_tiers ADD COLUMN max_inclusive BOOLEAN NOT NULL DEFAULT TRUE;
ALTER TABLE revenue_bonus_tiers ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';
CREATE TABLE custom_roles (                 -- Q-13: role & permission dinamis
    id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    code VARCHAR(50) NOT NULL, name VARCHAR(100) NOT NULL, permissions TEXT[] NOT NULL,
    is_system BOOLEAN NOT NULL DEFAULT FALSE, CONSTRAINT uq_role UNIQUE (tenant_id, code)
);
ALTER TABLE employees ADD COLUMN has_login BOOLEAN NOT NULL DEFAULT TRUE;
ALTER TABLE employees ADD COLUMN in_payroll BOOLEAN NOT NULL DEFAULT TRUE;
```

### Seed definisi (ringkas — daftar lengkap & default = PRD §17.3)
```sql
INSERT INTO setting_definitions (key, category, label, value_type, default_value, allowed_scopes, question_ref, requires_approval) VALUES
 ('feature.bpjs','feature','BPJS aktif','boolean','{"v":false}','{tenant,pay_group,employee}','Q-06',TRUE),
 ('feature.pph21','feature','PPh21 aktif','boolean','{"v":false}','{tenant,pay_group,employee}','Q-06',TRUE),
 ('feature.salary_holdback','feature','Gaji gantungan','boolean','{"v":false}','{pay_group,employee}','Q-14',TRUE),
 ('feature.bonus_catering','feature','Bonus catering','boolean','{"v":true}','{tenant,employee}','Q-15',TRUE),
 ('feature.early_arrival_incentive','feature','Insentif masuk < batas jam','boolean','{"v":true}','{pay_group,employee}',NULL,FALSE),
 ('feature.holiday_incentive','feature','Insentif tanggal merah','boolean','{"v":true}','{pay_group}',NULL,FALSE),
 ('feature.closure_pay','feature','Libur operasional dibayar sebagian','boolean','{"v":true}','{pay_group}',NULL,FALSE),
 ('weekly.total_hours_basis','weekly','Dasar Total Jam','enum','{"v":"counted_clock_in"}','{pay_group,employee}','Q-04',FALSE),
 ('weekly.earliest_count_time','weekly','Jam hitung paling awal','time','{"v":"05:00"}','{pay_group,employee}','Q-04',FALSE),
 ('bonus.outlet.allocation_method','bonus','Metode bagi pool','enum','{"v":"manual"}','{location}','Q-02',TRUE),
 ('overtime.rounding','overtime','Pembulatan lembur','enum','{"v":"ceil_hour"}','{pay_group,position,employee}','Q-16',FALSE),
 ('revenue.source','revenue','Sumber omset','multi_enum','{"v":["manual"]}','{location}','Q-11',FALSE),
 ('system.public_api','system','API publik','boolean','{"v":false}','{tenant}','Q-17',FALSE);  -- is_locked = TRUE
-- setting_values awal ABPS diisi per pay_group sesuai default PRD §17.3 dengan reason = 'Seed dari sheet lapangan Okt 2026'
```


---

---

## v2.2 — Aturan Persetujuan Dinamis (Approval Policies)

> `setting_definitions.requires_approval` (v2.1) **tidak lagi menjadi sumber kebenaran**. Kolom ini sekarang hanya *default seed*. Syarat persetujuan per pengaturan diatur Owner lewat `approval_policies`, yang juga berversi tanggal, per scope (urutan override sama), dan diaudit.

```sql
-- ============================================================
-- 37. APPROVAL POLICIES (syarat persetujuan per setting, berversi & per scope)
-- ============================================================
CREATE TABLE approval_policies (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    definition_id       BIGINT          REFERENCES setting_definitions(id),  -- NULL = berlaku untuk kategori
    category            VARCHAR(50),                    -- dipakai jika definition_id NULL (mis. 'bonus')
    scope_type          VARCHAR(20)     NOT NULL DEFAULT 'tenant',  -- 'tenant','pay_group','location','position','employee'
    scope_id            BIGINT,
    requires_approval   BOOLEAN         NOT NULL,
    approver_roles      VARCHAR(50)[]   NOT NULL DEFAULT '{owner}',  -- role/custom_roles.code yang boleh menyetujui
    min_approvers       SMALLINT        NOT NULL DEFAULT 1 CHECK (min_approvers BETWEEN 1 AND 5),
    approval_mode       VARCHAR(20)     NOT NULL DEFAULT 'any',      -- 'any' (N dari role mana pun) | 'each_role' (≥1 dari tiap role)
    allow_self_approval BOOLEAN         NOT NULL DEFAULT FALSE,      -- terkunci FALSE (lihat pengaman)
    expires_after_hours INT,                                          -- permintaan kedaluwarsa (NULL = tidak)
    applies_to_actions  VARCHAR(20)[]   NOT NULL DEFAULT '{create,update,supersede}',
    effective_from      DATE            NOT NULL,
    effective_to        DATE,
    status              VARCHAR(20)     NOT NULL DEFAULT 'active',   -- 'pending_approval','active','superseded','rejected'
    reason              TEXT            NOT NULL,
    created_by          BIGINT          NOT NULL REFERENCES employees(id),
    approved_by         BIGINT[],                                     -- penyetuju meta (Owner)
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT ck_ap_target CHECK ((definition_id IS NULL) <> (category IS NULL)),
    CONSTRAINT ck_ap_selfapprove CHECK (allow_self_approval = FALSE),
    CONSTRAINT ex_ap_no_overlap EXCLUDE USING gist (
        tenant_id WITH =, COALESCE(definition_id,0) WITH =, COALESCE(category,'') WITH =,
        scope_type WITH =, COALESCE(scope_id,0) WITH =,
        daterange(effective_from, effective_to, '[)') WITH &&
    ) WHERE (status = 'active')
);
CREATE INDEX idx_ap_lookup ON approval_policies(tenant_id, definition_id, scope_type, scope_id, effective_from DESC) WHERE status = 'active';

-- ============================================================
-- 38. APPROVAL REQUESTS & DECISIONS (multi-approver)
-- ============================================================
CREATE TABLE approval_requests (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT          NOT NULL REFERENCES tenants(id),
    target_type         VARCHAR(30)     NOT NULL,       -- 'setting_value' | 'approval_policy' | 'revenue_bonus_tier' | 'piece_rate_item' | ...
    target_id           BIGINT          NOT NULL,
    policy_snapshot     JSONB           NOT NULL,       -- policy yang berlaku saat request dibuat (roles, min, mode)
    requested_by        BIGINT          NOT NULL REFERENCES employees(id),
    status              VARCHAR(20)     NOT NULL DEFAULT 'pending',  -- 'pending','approved','rejected','expired','cancelled'
    expires_at          TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
CREATE TABLE approval_decisions (
    id                  BIGSERIAL PRIMARY KEY,
    approval_request_id BIGINT          NOT NULL REFERENCES approval_requests(id),
    approver_id         BIGINT          NOT NULL REFERENCES employees(id),
    approver_role       VARCHAR(50)     NOT NULL,
    decision            VARCHAR(10)     NOT NULL,       -- 'approve','reject'
    note                TEXT,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_decision UNIQUE (approval_request_id, approver_id)
);

-- setting_audit_logs diperluas agar mencatat juga kebijakan persetujuan
ALTER TABLE setting_audit_logs ADD COLUMN target_type VARCHAR(30) NOT NULL DEFAULT 'setting_value'; -- 'setting_value'|'approval_policy'|'approval_request'
ALTER TABLE setting_audit_logs ADD COLUMN approval_policy_id BIGINT REFERENCES approval_policies(id);
ALTER TABLE setting_audit_logs ADD COLUMN approval_request_id BIGINT REFERENCES approval_requests(id);
-- action tambahan: 'policy_create','policy_supersede','request','approve','reject','expire'

-- setting_values.approved_by → digantikan approval_requests/decisions (kolom tetap untuk ringkasan)
ALTER TABLE setting_values ADD COLUMN approval_request_id BIGINT REFERENCES approval_requests(id);
```

### Pengaman (guardrail) — ditegakkan di DB + aplikasi
| # | Pengaman |
|---|---|
| G1 | **Meta-approval:** setiap perubahan `approval_policies` wajib disetujui **Owner** (`min_approvers ≥ 1`, role `owner`). Aturan ini sendiri adalah policy terkunci (`meta.approval_policy`, `is_locked = TRUE`), tidak bisa dimatikan dari UI. |
| G2 | **Tanpa self-approval:** pembuat ≠ penyetuju (CHECK + validasi). Jika Owner yang mengubah policy dan hanya ada satu Owner, perubahan wajib **dual-control**: Owner + HR Admin (atau Owner kedua). Tidak ada jalur tanpa jejak. |
| G3 | **Lantai minimum (floor):** key yang ditandai `approval_floor = TRUE` (feature.bpjs, feature.pph21, feature.salary_holdback, bonus.*, rate.*, meta.*) tidak bisa diturunkan ke `requires_approval = FALSE` kecuali disetujui **2 penyetuju** (Owner + 1 lainnya). Policy tetap tercatat dan bisa dikembalikan. |
| G4 | **Tidak retroaktif:** `effective_from` policy ≥ hari ini; request yang sudah dibuat memakai `policy_snapshot` saat dibuat. |
| G5 | **Append-only:** audit log tidak bisa UPDATE/DELETE (REVOKE + trigger). Mematikan approval selalu menghasilkan baris audit `policy_supersede` dengan alasan wajib. |
| G6 | **Terkunci permanen:** `system.public_api` = off, `is_locked = TRUE`, tidak punya policy, tidak bisa diubah oleh role apa pun dari UI/API (hanya migration + code review). |
| G7 | **Notifikasi:** setiap penonaktifan approval atau penurunan `min_approvers` mengirim notifikasi ke semua Owner & HR Admin. |

```sql
ALTER TABLE setting_definitions ADD COLUMN approval_floor BOOLEAN NOT NULL DEFAULT FALSE;
COMMENT ON COLUMN setting_definitions.requires_approval IS 'v2.2: hanya default seed; sumber kebenaran = approval_policies';
```

### Seed default approval policies (ABPS)
```sql
-- effective_from = tanggal go-live; reason = 'Default v2.2 disetujui Pak Irvan 24 Okt 2026'
-- ON, approver owner, min 1:
--   feature.bpjs, feature.pph21, bpjs.*, pph21.*          (approval_floor)
--   kategori 'bonus' (bonus.outlet.*, feature.bonus_catering, revenue_bonus_tiers, bonus_position_weights)  (floor)
--   feature.salary_holdback, holdback.*                    (floor)
--   kategori 'rate' (rate.*, payroll.salary_split, piece_rate.items, part_time.visit_fee, overtime.flat_rate)  (floor)
--   rbac.role_permissions                                  (floor)
-- ON, approver owner+hr_admin, min 2, mode each_role:
--   meta.approval_policy (terkunci)
-- OFF (cukup audit): kategori attendance, shift, slip, period, weekly.*, deduction.*, overtime.* selain tarif
-- LOCKED, tanpa policy: system.public_api
```


---

## Seed Data ABPS (berbasis data lapangan Okt 2026)

> Aturan seed: **tanpa NIK, KK, no. rekening, no. HP, email asli**. Email login memakai placeholder `<kode>@abps.local` dan password wajib diganti saat login pertama. Nominal gaji diambil dari sheet *Gaji Bulanan Sept/Okt 2026* & *Gaji Mingguan Okt 2026*. Seed hanya untuk environment staging; data produksi diimpor via fitur import Excel.

```sql
-- TENANT
INSERT INTO tenants (name, slug, primary_color, secondary_color, font_heading, font_body, slip_header_color, timezone)
VALUES ('Ayam Bakar Pucuk Singkong', 'abps', '#1F8802', '#A76A02', 'Cabin', 'Albert Sans', '#1F8802', 'Asia/Jakarta');
-- Catatan: prototipe menampilkan "PT Pucuk Singkong Sejahtera"; nama badan hukum resmi perlu dikonfirmasi (Q-12).

-- LOCATIONS (koordinat BELUM diketahui → placeholder 0,0, wajib diisi sebelum go-live, Q-09)
INSERT INTO locations (tenant_id, name, latitude, longitude, radius_meter, allow_outside_with_reason, require_photo) VALUES
  (1, 'Outlet Baros',  0, 0, 100, TRUE, TRUE),
  (1, 'Dapur Pusat',   0, 0, 100, TRUE, TRUE);
  -- Kantor/Admin pusat diasumsikan satu lokasi dengan Dapur Pusat (Q-09)

-- DEPARTMENTS
INSERT INTO departments (tenant_id, name, code) VALUES
  (1, 'Operasional Outlet', 'OPS'),
  (1, 'Produksi Dapur Pusat', 'PRD'),
  (1, 'Kantor Pusat', 'HO');

-- PAY GROUPS
INSERT INTO pay_groups (tenant_id, code, name, pay_frequency, week_start_dow, payday_rule) VALUES
  (1, 'WEEKLY_PUSAT',   'Produksi Dapur Pusat (mingguan)', 'weekly', 6, 'weekly:friday'),     -- Sabtu–Jumat
  (1, 'MONTHLY_OUTLET', 'Crew Outlet (bulanan)',            'monthly', NULL, 'monthly:3'),     -- jadwal Okt: "3 / payday"
  (1, 'MONTHLY_KANTOR', 'Kantor Pusat (bulanan)',           'monthly', NULL, 'monthly:3'),
  (1, 'PART_TIME',      'Part-time / Freelance',            'monthly', NULL, NULL);

-- PAY RULES (nilai berlaku Okt 2026)
INSERT INTO pay_rules (tenant_id, pay_group_id, effective_from, divisor_days_per_month, standard_hours_per_day,
  earliest_count_time, early_arrival_incentive, early_arrival_before,
  overtime_mode, overtime_flat_rate, overtime_threshold_minutes, overtime_grace_minutes, overtime_rounding,
  late_deduction_per_minute, late_deduction_mode, izin_deduction_basis, sakit_deduction_basis, sakit_flat_per_day) VALUES
  (1, 1, '2026-10-01', 26, 9, '05:00', 10000, '05:00', 'flat_per_hour', 10000, 540, 30, 'ceil_hour', 0,   'none',           'none',                    'none',                         NULL),
  (1, 2, '2026-09-01', 26, 9, NULL,    0,     NULL,    'flat_per_hour', 10000, 540, 0,  'input_hours', 500, 'per_minute',   'total_salary_div_days',   'variable_allowance_div_days',  NULL),
  (1, 3, '2026-09-01', 26, 9, NULL,    0,     NULL,    'flat_per_hour', 17000, 540, 0,  'input_hours', 0,   'prorata_hourly', 'variable_allowance_div_days', 'flat',                       50000);

-- SHIFT TEMPLATES (jadwal outlet Okt 2026)
INSERT INTO shift_templates (tenant_id, name, code, start_time, end_time, break_duration_minutes, tolerance_late_minutes, work_days) VALUES
  (1, 'Pagi (P1)',  'P1',  '06:00', '16:00', 30, 0, '1,2,3,4,5,6,7'),
  (1, 'Siang (P2)', 'P2',  '11:30', '21:30', 30, 0, '1,2,3,4,5,6,7'),
  (1, 'Middle',     'MID', '09:30', '19:30', 30, 0, '1,2,3,4,5,6,7'),
  (1, 'Shift 3',    'S3',  '12:30', '21:30', 30, 0, '1,2,3,4,5,6,7'),
  (1, 'Produksi Pusat', 'PRD', '05:00', '14:00', 0, 0, '1,2,3,4,5,6,7');
  -- toleransi telat crew = 0 menit (sheet memotong Rp500 sejak menit pertama; Q-10)
  -- break 30 mnt: tertulis di jadwal versi Juni ("10 jam, istirahat 30 menit"); jam produksi pusat 05:00–14:00 = ASUMSI dari clamp 05:00 + 9 jam (Q-04)

-- STATION CODE (dipakai di shift_assignments.station_code)
--   ck = Checker, kit/kc = Kitchen, ks = Kasir, se = Server, bk = Bakar

-- SALARY COMPONENTS
INSERT INTO salary_components (tenant_id, name, code, component_type, is_taxable, calculation_basis, is_system) VALUES
  (1, 'Gaji Pokok',                     'POKOK',        'earning_fixed',      FALSE, 'fixed', TRUE),
  (1, 'Tunjangan Tidak Tetap (Kehadiran & Transport)', 'TUNJ_TT', 'earning_fixed', FALSE, 'fixed', TRUE),
  (1, 'Tunjangan Jabatan',              'TUNJ_JAB',     'earning_fixed',      FALSE, 'fixed', TRUE),
  (1, 'Upah Jam Kerja (mingguan)',      'UPAH_JAM',     'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Lembur',                         'LEMBUR',       'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Insentif Masuk < 05:00',         'INS_PAGI',     'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Insentif Tanggal Merah',         'INS_TGLMERAH', 'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Bonus Omset Outlet',             'BONUS_OUTLET', 'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Bonus Catering',                 'BONUS_CATER',  'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Bonus Akhir Tahun',              'BONUS_AT',     'earning_variable',   FALSE, 'fixed', FALSE),
  (1, 'Transport',                      'TRANSPORT',    'earning_variable',   FALSE, 'fixed', FALSE),
  (1, 'Borongan / Per Tugas',           'BORONGAN',     'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Fee Kunjungan',                  'VISIT',        'earning_variable',   FALSE, 'fixed', TRUE),
  (1, 'Potongan Telat',                 'POT_TELAT',    'deduction_variable', FALSE, 'fixed', TRUE),
  (1, 'Potongan Izin',                  'POT_IZIN',     'deduction_variable', FALSE, 'fixed', TRUE),
  (1, 'Potongan Sakit',                 'POT_SAKIT',    'deduction_variable', FALSE, 'fixed', TRUE),
  (1, 'Absent Prorate',                 'POT_PRORATE',  'deduction_variable', FALSE, 'fixed', TRUE),
  (1, 'Kasbon',                         'KASBON',       'deduction_variable', FALSE, 'fixed', TRUE),
  (1, 'Gaji Gantungan (ditahan)',       'GANTUNGAN',    'deduction_variable', FALSE, 'fixed', FALSE);
-- BPJS_KES, BPJS_TK, PPH21 TIDAK di-seed untuk ABPS (tidak ada di slip lapangan). Tersedia sebagai komponen opsional (Q-06).

-- PIECE RATE ITEMS (freelance pusat)
INSERT INTO piece_rate_items (tenant_id, code, name, unit, rate, effective_from) VALUES
  (1, 'KURIR_ANTAR_BAHAN',   'Kurir antar bahan',      'task', 20000, '2026-01-01'),
  (1, 'KURIR_SAMPAH_TPS',    'Buang sampah ke TPS',    'task', 20000, '2026-01-01'),
  (1, 'KURIR_ANTAR_PESANAN', 'Antar pesanan',          'task', 20000, '2026-01-01'),
  (1, 'METIK_CENGEK',        'Metik cengek (cabai rawit)', 'kg', 2500, '2026-01-01'),
  (1, 'METIK_BAPUT',         'Kupas bawang putih',     'kg',   3000, '2026-01-01');
  -- effective_from = placeholder; tanggal mulai tarif belum diketahui

-- REVENUE BONUS TIERS (pesan WA "Bonus Outlet", Kamis 8 Jan)
INSERT INTO revenue_bonus_tiers (tenant_id, bonus_type, min_net_revenue, max_net_revenue, percent_of_net, reference_headcount, effective_from) VALUES
  (1, 'outlet', 0,         349999999, 0,   12, '2026-01-08'),
  (1, 'outlet', 350000000, 400000000, 0.5, 12, '2026-01-08'),
  (1, 'outlet', 400000001, NULL,      1.0, 12, '2026-01-08');
```

### Contoh karyawan & skema gaji (Okt 2026, nama panggilan saja)

| Kode | Nama | Pay group | Jabatan / station | Skema | Nominal |
|---|---|---|---|---|---|
| P001 | Lilis | WEEKLY_PUSAT | Produksi | harian÷9 per jam | Rp130.000/hari (≈14.444/jam) |
| P002 | Titin ("Teh Ade") | WEEKLY_PUSAT | Produksi | harian÷9 | Rp120.000/hari |
| P003 | Yuyun | WEEKLY_PUSAT | Produksi | harian÷9 | Rp100.000/hari |
| P004 | Neni ("Teh Leni") | WEEKLY_PUSAT | Produksi | harian÷9 | Rp90.000/hari |
| P005 | Teh Titin | PART_TIME | Freelance | per jam + borongan + kurir | Rp9.000/jam |
| — | Dahlia | MONTHLY_KANTOR | Manager Operasional | 60/25/15 | Rp5.000.000 |
| — | Yuni | MONTHLY_KANTOR | Admin | 60/40 | (lihat sheet) |
| — | Meylinda | MONTHLY_KANTOR | Purchasing | 2,0jt → 2,3jt pertengahan bulan (prorata /26) | |
| CW0xx | Nasir | MONTHLY_OUTLET | Leader | 50/30/20 | (lihat sheet) |
| CW0xx | Vanesa, Airi | MONTHLY_OUTLET | Kasir 1 / Kasir 2 | 50/50 | |
| CW0xx | Refa | MONTHLY_OUTLET | Kasir & Server | 50/50, masa percobaan 1,8jt → 2jt setelah 3 bln | |
| CW0xx | Iman, Rafli | MONTHLY_OUTLET | Checker | 50/50 | |
| CW0xx | M Iqbal, Reyhan | MONTHLY_OUTLET | Server | 50/50 (Iqbal 1,8jt percobaan) | |
| CW0xx | Yuli, Herni | MONTHLY_OUTLET | Kitchen | 50/50 | |
| CW0xx | Hesti | MONTHLY_OUTLET | Kitchen & Server | 50/50 (1,8jt percobaan) | |
| — | Tri Wulandari | PART_TIME | Content Specialist | Rp1.200.000 + Rp50.000/visit | |

> Pembagian 50/50 = 50% Gaji Pokok + 50% Tunjangan Tidak Tetap; 50/30/20 = Pokok/Tunj. Tidak Tetap/Tunj. Jabatan. Disimpan sebagai `employee_salary_components` (nominal absolut, effective-dated) — **bukan** persentase hardcode.
> Tarif "planing" (produksi 16/14/13/12 rb/jam, admin 1,75jt/1,7jt + 30rb/hari, Manager Produksi 2jt + 100rb/hari) **belum** dipakai sebagai seed sampai dikonfirmasi berlaku (Q-08).

---

## Catatan Index Strategy

### Tabel `attendances` (volume tertinggi)
```
- Composite (employee_id, attendance_date DESC)     → rekap per karyawan
- Composite (tenant_id, attendance_date DESC)       → dashboard harian HR
- Partial (fraud_flag = TRUE)                       → monitoring keamanan
- Partial (late_minutes > 0)                        → laporan keterlambatan
```

### Tabel `payroll_items`
```
- Composite (employee_id, payroll_run_id DESC)      → history slip gaji karyawan
- Composite (payroll_run_id, status)                → progress payroll run
```

### Tabel `employees`
```
- GIN trgm pada full_name                           → search karyawan
- Composite (tenant_id, role)                       → role-based query
- Partial WHERE deleted_at IS NULL                  → soft delete efficiency
```

### Tabel `shift_assignments`
```
- Composite (employee_id, effective_date, end_date) → jadwal aktif karyawan
```

---

## Estimasi Volume Data (3 Tahun)

| Tabel | Baris / Tahun | 3 Tahun |
|---|---|---|
| employees | ABPS ±25 aktif (12 outlet + 5 pusat + kantor + part-time) | <100 |
| attendances | 25 × 360 hari (outlet buka setiap hari) | ~27.000 / tenant |
| shift_assignments | 500 × 12 siklus | ~18.000 / tenant |
| payroll_items | 5 × 52 minggu + 20 × 12 bulan | ~1.500 / tenant |
| payroll_item_details | 18.000 × 8 komponen | ~144.000 / tenant |
| audit_logs | ~2M / tahun | ~6M |

> Dengan indexing di atas, query rekap kehadiran 100K baris selesai < 50ms di PostgreSQL 15+ dengan hardware standar (8-core, 16GB RAM).
