# CHANGELOG & Analisa — Revisi Dokumen HRD ABPS v2.0
**Tanggal:** 24 Oktober 2026 · **Disusun untuk:** Pak Irvan
**Sumber analisa:** 4 workbook (Gaji Mingguan Dapur Pusat, Gaji Bulanan Outlet/Kantor, Jadwal & Kas Outlet, Master Karyawan "saat ini"/"planing"), slip PDF, gambar WA "Bonus Outlet", prototipe https://digihrd.vercel.app/ (semua halaman + API).

> Data pribadi (NIK, KK, rekening, HP) **tidak** disalin ke dokumen mana pun.

---

## 1. Temuan Lapangan

### 1.1 Struktur organisasi & gaji
- Ada 3 unit: **Outlet Baros** (±12 crew), **Dapur Pusat/Produksi** (4 tetap + 1 freelance), **Kantor Pusat** (Admin, Manager Ops, Purchasing) + part-time (Content Specialist).
- **Dua frekuensi gaji**: produksi pusat **mingguan Sabtu–Jumat**; outlet & kantor **bulanan**.
- **Tidak ditemukan BPJS maupun PPh21** di sheet atau slip mana pun.

### 1.2 Produksi pusat (mingguan)
- Upah per jam = gaji harian ÷ 9 (Okt: 130rb / 120rb / 100rb / 90rb). Des–Jan memakai tarif jam langsung 12,5rb / 11rb / 9,5rb / 8,5rb.
- Jam hitung = MAX(jam masuk, 05:00); masuk < 05:00 → insentif Rp10.000.
- Upah dihitung per menit (total jam × 24 × tarif, di Excel).
- Lembur: > 9 jam, kelebihan ≤ 30 menit diabaikan, sisanya ROUNDUP ke jam × Rp10.000.
- Sejak ±Agustus ada kolom "Hari Kerja Full 9 Jam" (Jumat) yang dibayar flat gaji harian.
- Insentif tanggal merah Rp50.000; bonus akhir tahun (27 Des–2 Jan); jam hari Minggu kadang dibayar terpisah.
- Gaji bersih = Gaji + Penambahan − Kasbon, transfer BCA.
- Freelance: Rp9.000/jam + metik cengek 2.500/kg, baput 3.000/kg + kurir 20.000/tugas.

### 1.3 Outlet & kantor (bulanan)
- Gaji dipecah: crew 50/50 (Pokok / Tunj. Tidak Tetap), Leader 50/30/20 (+ Jabatan), Admin 60/40, Manager Ops 60/25/15.
- Basis 26 hari; karyawan baru/percobaan & kenaikan tengah bulan diprorata (÷26). Percobaan 1,8 jt → rencana 2 jt setelah 3 bulan.
- Lembur crew 10rb/jam (pecahan 0,5 jam), kantor 17rb/jam.
- Telat crew Rp500/menit; kantor prorata gaji/26/9 per jam.
- Izin crew = gaji/26 per hari; kantor = tunjangan/26. Sakit crew = tunjangan/26; kantor Rp50.000/hari.
- Aturan **berubah dari waktu ke waktu** (Maret 2026: izin 80rb/hari, sakit 50rb/hari) → wajib berversi tanggal.
- Kolom BONUS (180rb / 80rb), transport, kasbon (termasuk "Kasbon Abah"), ABSENT PRORATE.
- Maret: libur Lebaran 18–28 Mar dibayar 50%; ada "GAJI GANTUNGAN" Rp450rb × 2 minggu.
- Template slip "SLIP GAJI KARYAWAN AYAM BAKAR PUCUK SINGKONG", penandatangan HR PUCUK SINGKONG. Sheet "Payslip Kantor" mengandung error `#REF!`.

### 1.4 Operasional outlet
- Jam buka 07.00–21.00. Shift P1 06.00–16.00, P2 11.30–21.30, Middle 09.30–19.30, Shift 3 12.30/11.30–21.30, plus jam custom.
- Station ck/kit/ks/se dirotasi per shift; status off / off izin / off sakit; tanggal 3 ditandai "payday".
- Tukar shift wajib ada pengganti + konfirmasi Leader. Izin/sakit diajukan malam sebelumnya atau ≥ 6 jam sebelum shift; surat sakit boleh H+1.
- Bonus outlet (WA 8 Jan, kondisi 12 karyawan): < 350 jt → 0; 350–400 jt → 0,5%; > 400 jt → 1% dari omset bersih (kotor − diskon − potongan ojol).
- Kas outlet memakai MOKA / Runchise / Kledo, tetapi sheet tidak memuat omset bersih, diskon, atau potongan ojol.

## 2. Gap Prototipe vs Kebutuhan

| Area | Prototipe saat ini | Kebutuhan ABPS | Dokumen |
|---|---|---|---|
| Auth | "Mode HR Admin (Tanpa Login)", `/login` 404 | Login wajib (karyawan via nomor + PIN) | TRD §6.4–6.5 |
| Keamanan API | `/api/*` dapat dibaca publik, termasuk NIK/HP/email | 401 tanpa sesi; DTO tanpa PII | TRD §6.4, CLAUDE #9 |
| Data | Seed dummy PT Maju, Pabrik Cimahi, dll. | Data ABPS (Outlet Baros, Dapur Pusat) | ERD Seed |
| Periode payroll | Hanya bulanan, unik per bulan | Mingguan Sabtu–Jumat + bulanan | ERD `pay_groups`, `payroll_runs` |
| Lembur | Multiplier 1,5 | Flat 10rb/17rb per jam, aturan 9 jam + 30 mnt | ERD `pay_rules`, TRD §8.3.1 |
| Komponen | BPJS, PPh21, Tunj. Makan | Pokok / Tunj. TT / Jabatan, kasbon, insentif, bonus, borongan | ERD seed komponen |
| Kasbon / borongan / bonus omset / gantungan | Tidak ada | Wajib | ERD tabel 24–30 |
| Jadwal | Grid mingguan Senin–Minggu, shift dummy | Grid bulanan + station + off/izin/sakit, P1/P2/MID/S3 | PRD §6 |
| Cuti | Annual/Sick/Emergency/Unpaid | Izin / Sakit (+ surat H+1), tidak ada saldo cuti | PRD §3, §6.4 |
| Slip PDF | Route tidak ditemukan | Template slip bulanan & mingguan ABPS | PRD §9, AGENTS |
| Laporan | Kolom BPJS/PPh21 | Kolom seperti sheet lama | PRD §12 |
| Dashboard | Ada di menu, tanpa route khusus | Ringkasan hadir/telat, run, kasbon, omset | PRD §12 |
| Penamaan API | `/api/payroll-runs` (dokumen v1: `/api/payroll/runs`) | Dokumen diselaraskan ke prototipe | AGENTS, TRD §8.3.8 |
| Branding | #1f8802 / #a76a02, Cabin/Albert Sans, "PT Pucuk Singkong Sejahtera" | Dipertahankan; nama badan hukum dikonfirmasi | PRD §10 |

## 3. Perubahan per File

**ERD.md (v1.0 → v2.0)**
- Header & diagram baru.
- `employees`: tambah `pay_group_id`, `daily_rate`, `probation_until`; email opsional; status `part_time`.
- `positions.payroll_type` diganti menjadi `monthly_split` / `weekly_hourly` / `freelance_hourly` / `part_time_visit`.
- `shift_templates.code`; `shift_assignments.station_code` & `day_status`.
- `overtime_requests` → `rate_mode`, `flat_rate`, `payable_hours`.
- `payroll_runs` → `pay_group_id`, `pay_frequency`, `period_start/end`; unik (tenant, pay_group, period_start); status disamakan dengan prototipe.
- `payroll_items` → izin/sakit/full-9h/early/holiday, kasbon, bonus, holdback; `payroll_item_details` → quantity & unit_rate.
- Tabel baru 21–30: pay_groups, pay_rules, holidays, cash_advances (+repayments), piece_rate_items, piece_work_entries, visit_logs, outlet_revenues, revenue_bonus_tiers, bonus_pools/allocations, salary_holdbacks.
- Seed dummy diganti seed ABPS tanpa PII; estimasi volume disesuaikan.

**PRD.md**
- Ditulis ulang untuk ABPS: ruang lingkup, peta peran, presensi 2 lokasi, jadwal outlet, aturan telat per pay group.
- Payroll mingguan/bulanan/freelance/part-time, kasbon, borongan, bonus omset, slip format ABPS.
- Acceptance criteria berbasis paralel run dengan sheet; daftar Q.

**TRD.md**
- Header v2.0.
- §6.4 temuan keamanan & perbaikan, §6.5 login PIN.
- §8.3 calculator ABPS (modul per skema, test fixture dari sheet), §8.3.7 kasbon/borongan/holdback, §8.3.8 lifecycle run.
- Struktur `lib/payroll` diperbarui; BPJS/PPh21 di balik feature flag.

**AGENTS.md**
- Prinsip baru: jangan mengarang aturan, wajib fixture, dilarang memakai PII.
- Registri payroll dipecah per skema; modul baru Kasbon, Borongan, Omset & Bonus, Pengaturan Payroll.
- Path API diselaraskan ke prototipe; urutan prioritas pengerjaan.

**CLAUDE.md**
- Identitas ABPS + glosarium domain.
- Aturan absolut baru #8 (tanpa angka payroll hardcode), #9 (PII), #10 (run approved immutable), #11 (BPJS/PPh21 feature flag).

## 4. Asumsi yang Dipakai (perlu divalidasi)
- A1. Periode bulanan = bulan kalender; payday tgl 3 bulan berikutnya.
- A2. Kantor pusat berlokasi sama dengan Dapur Pusat.
- A3. Jam produksi pusat 05:00–14:00 (dari clamp 05:00 + 9 jam).
- A4. Toleransi telat crew 0 menit.
- A5. Pool bonus Sept = 0,5% × 352 jt (inferensi dari total 1,76 jt), dibagi manual.
- A6. Bonus Admin 620rb = bonus catering 0,5%.
- A7. Tarif yang berlaku = sheet "saat ini" / slip Sept–Okt, bukan "planing".

## 5. Keputusan Konfigurasi (v2.1)

Sesuai arahan Pak Irvan, tidak ada pertanyaan yang dijawab dengan satu pilihan. Setiap Q/A menjadi pengaturan dinamis yang bisa di-on/off-kan, punya beberapa opsi metode, angkanya bisa diubah kapan saja, berlaku per tanggal (effective_from/to), dan bisa di-override per kelompok gaji, lokasi, jabatan, atau karyawan. Semua perubahan tercatat di audit log. Default awal = praktik di sheet. Detail key, tipe, dan scope: `PRD.md` §17.3.

| Q/A | Hal | Opsi yang tersedia | Default awal (berdasarkan sheet) | Siapa yang bisa ubah | Persetujuan default (v2.2) |
|---|---|---|---|---|---|
| Q-01 | Tier bonus outlet (batas & persen) (`bonus.outlet.tiers`) | daftar tier bebas {min,max,percent}; boundary per tier: `inclusive`/`exclusive`; headcount_rule: `fixed`/`scale_by_headcount`/`ignore` | <350jt 0% · 350–400jt 0,5% · >400jt 1%; 400jt = inklusif ke 0,5%; headcount ref 12 = `ignore` | HR Admin + Owner | On · Owner · floor |
| Q-02 | Pembagian pool bonus (`bonus.outlet.allocation_method`) | `equal` (rata) · `attendance_days` (proporsional hari hadir) · `position_weight` (bobot per jabatan) · `attendance_x_weight` · `manual` · `fixed_amount_with_penalty` (nominal tetap, dikurangi jika izin/sakit) | `manual` (praktik Sept: 180rb/80rb); `bonus.outlet.eligible_positions` = semua crew + Manager Ops | HR Admin + Owner | On · Owner · floor |
| Q-03 | Kerja hari Minggu pusat (`weekly.sunday_policy`) | `normal_hourly` · `separate_hourly_rate` (tarif sendiri) · `multiplier` · `flat_per_day` · `excluded` | `normal_hourly`, dibayar di baris terpisah "Gaji di Hari Minggu/Jam" | HR Admin | Off (audit saja) |
| Q-04 | Dasar Total Jam pusat (`weekly.total_hours_basis`) | `counted_clock_in` (mulai 05:00) · `raw_clock_in` (jam masuk asli) | `counted_clock_in`; `weekly.earliest_count_time` = 05:00 | HR Admin | Off (audit saja) |
| Q-05 | Hari "Full 9 Jam" dibayar flat (`weekly.full_day_flat`) | on/off; `days_of_week` (multi), `min_minutes`, `pay`: `daily_rate`/`hourly` | on; Jumat; min 0 menit; `daily_rate` | HR Admin | Off (audit saja) |
| Q-06 | BPJS Kes/TK & PPh21 (`feature.bpjs / feature.pph21`) | on/off masing-masing; persen & plafon per program; metode PPh21 `TER`/`none`; ditanggung `employee`/`company` | off / off | HR Admin | On · Owner · floor |
| Q-07 | Periode & payday (`period.monthly_cutoff · payday`) | cutoff 1–28 atau `calendar`; payday tgl N / `last_working_day`; mingguan `week_start_dow` 0–6, payday dow | bulanan `calendar`, payday tgl 3; mingguan Sabtu–Jumat, payday Jumat | HR Admin | Off (audit saja) |
| Q-08 | Tarif berlaku (saat ini vs planing) (`rate.* (daily_rate, hourly_rate, salary components)`) | nilai per karyawan/jabatan dengan effective_from; rencana disimpan sebagai versi `scheduled` (tanggal masa depan) | tarif "saat ini"/slip Sept–Okt; "planing" = draft belum aktif | HR Admin + Owner | On · Owner · floor |
| Q-09 | Lokasi, radius, foto, kantor=dapur (`attendance.geofence`) | lat/lng, radius m, `require_photo` on/off, `allow_outside_with_reason` on/off, `mock_gps_action`: `block`/`flag`; karyawan boleh multi-lokasi on/off | radius 100 m, foto on, luar radius perlu alasan, mock = block; koordinat kosong → clock-in GPS non-aktif sampai diisi | HR Admin | Off (audit saja) |
| Q-10 | Toleransi telat & izin mendadak (`attendance.late_tolerance · leave.notice_rule`) | toleransi menit; `late_deduction_mode`: `per_minute`/`prorata_hourly`/`per_hour`/`flat_cut`/`tiered`/`none`; batas notice jam; sanksi notice: `none`/`flag`/`flat`/`as_alpha`; surat sakit H+N | crew 0 mnt, 500/mnt; kantor prorata; notice 6 jam/malam sebelum → `flag`; surat sakit H+1 | HR Admin | Off (audit saja) |
| Q-11 | Sumber omset bersih (`revenue.source`) | `manual` · `import_moka` · `import_runchise` · `import_kledo` · `import_ojol` (CSV/XLSX); `net_formula` komponen pengurang (diskon, ojol, pajak, retur) on/off; `require_verification` on/off | `manual`; pengurang = diskon + potongan ojol; verifikasi on (Owner) | Manager Ops input, Owner verifikasi | Off (audit saja) |
| Q-12 | Nama di aplikasi/slip (`tenant.display_name · slip.signatory`) | nama aplikasi, nama di slip, penandatangan, logo | app "Ayam Bakar Pucuk Singkong"; slip "SLIP GAJI KARYAWAN AYAM BAKAR PUCUK SINGKONG"; ttd "HR PUCUK SINGKONG" | HR Admin | Off (audit saja) |
| Q-13 | Peran & siapa masuk sistem (`rbac.role_permissions · employee.system_user`) | role → permission (bisa custom role); per karyawan `has_login` on/off, `in_payroll` on/off | default matriks PRD §4; Manager Produksi & kurir: in_payroll on, has_login off | HR Admin + Owner | On · Owner · floor |
| Q-14 | Gaji gantungan (`feature.salary_holdback`) | on/off; nominal `fixed`/`percent`; frekuensi per run; `release_rule`: `manual`/`after_n_runs`/`on_resign`/`date`; berlaku siapa (pay_group/employee) | off (riwayat Maret 450rb×2 bisa dicatat manual) | HR Admin + Owner | On · Owner · floor |
| Q-15 | Bonus catering (`feature.bonus_catering`) | on/off; persen; basis omset `catering_net`/`catering_gross`/`manual`; penerima (daftar karyawan/jabatan); alokasi seperti Q-02 | on, 0,5%, basis `manual`, penerima Admin (asumsi A6) | HR Admin + Owner | On · Owner · floor |
| Q-16 | Lembur outlet/kantor/pusat (`overtime.*`) | `mode`: `flat_per_hour`/`multiplier`/`tiered`; tarif; `threshold_minutes`; `grace_minutes`; `rounding`: `none`/`ceil_hour`/`floor_half`/`nearest_half`/`ceil_half`; `source`: `auto_from_attendance`/`request_approval`/`both`; approver role; min durasi | pusat: flat 10rb, 540 mnt, grace 30, `ceil_hour`, auto. Crew: flat 10rb, `nearest_half`, `request_approval` (Leader). Kantor: flat 17rb, `request_approval` (Manager) | HR Admin | Off; tarif flat: On · Owner · floor |
| Q-17 | Data dummy & API publik prototipe (`system.demo_data · system.public_api`) | `demo_data` on/off (hapus seed); `public_api` selalu off di production (tidak bisa diubah via UI) | demo off saat go-live; public_api off (hard rule, bukan setting) | Developer/Super Admin | 🔒 terkunci mati |
| A1 | Periode bulanan & payday (`period.monthly_cutoff`) | lihat Q-07 | calendar, tgl 3 | HR Admin | Off (audit saja) |
| A2 | Kantor = Dapur Pusat (`location.mapping`) | `same_location`/`separate_location` | `same_location` | HR Admin | Off (audit saja) |
| A3 | Jam produksi pusat (`shift.production_template`) | start/end/break per template, berversi | 05:00–14:00, break 0 | HR Admin | Off (audit saja) |
| A4 | Toleransi telat crew (`attendance.late_tolerance`) | lihat Q-10 | 0 menit | HR Admin | Off (audit saja) |
| A5 | Pool bonus Sept (`bonus.outlet.allocation_method`) | lihat Q-01/Q-02 | manual; pool dihitung dari omset input | HR Admin + Owner | On · Owner · floor |
| A6 | Bonus Admin = catering (`feature.bonus_catering`) | lihat Q-15 | on untuk Admin | HR Admin + Owner | On · Owner · floor |
| A7 | Tarif saat ini vs planing (`rate.*`) | lihat Q-08 | saat ini | HR Admin + Owner | On · Owner · floor |

Setting tambahan yang juga dinamis: insentif masuk < 05:00, insentif tanggal merah, Lebaran/libur operasional dibayar sebagian, potongan izin/sakit/alpa, divisor 26 & 9 jam, metode prorata, pecahan gaji, kebijakan kasbon, tarif borongan, fee kunjungan, bonus akhir tahun/transport, template slip (lihat PRD §17.3).

Kolom *Persetujuan default* bukan nilai tetap: Owner bisa mengubahnya per pengaturan, per scope, dan per tanggal (PRD §18). Perubahan tersebut wajib disetujui Owner + HR Admin dan tercatat di audit log.

## 6. Changelog v2.1 (24 Okt 2026)
- **ERD:**
  - tabel 31–36: `setting_definitions`, `setting_options`, `setting_values` (scope + effective_from/to + status + larangan overlap), `setting_audit_logs` (append-only), `payroll_run_snapshots`
  - `bonus_position_weights`, `revenue_imports`, `custom_roles`
  - kolom baru: inklusif/eksklusif di tier bonus, `has_login`/`in_payroll`, run adjustment
  - `pay_rules` menjadi lapisan kompatibilitas
- **PRD:**
  - §16 diganti menjadi Keputusan Konfigurasi
  - §17 baru: layar Pengaturan Aturan, urutan override, matriks Q/A → opsi → default, aturan bonus outlet dinamis, permission, acceptance criteria
- **TRD:** §15 baru:
  - resolver dengan override 5 level
  - query satu round-trip
  - validasi (zod, lintas setting, larangan retroaktif, four-eyes)
  - preview dampak
  - snapshot dan immutability setelah approve
  - API settings dan test wajib
  - calculator §8.3 kini memakai strategi per opsi
- **AGENTS:** prinsip "semua lewat resolver" dan "jangan memilih jawaban bisnis"; modul Rule Engine; prioritas: rule engine sebelum calculator.
- **CLAUDE:** aturan #8 diperluas (angka + toggle + metode); aturan baru #12 (versi + audit), #13 (payroll baca snapshot), #14 (ambiguitas → opsi).
- Prototipe tidak diubah, dan tidak ada data pribadi yang dimasukkan.

## 7. Changelog v2.2 (24 Okt 2026) — Persetujuan dinamis (disetujui Pak Irvan)
- **ERD:**
  - tabel 37–38: `approval_policies` (on/off, role penyetuju, min penyetuju, mode any/each_role, kedaluwarsa, scope, effective_from/to, status, alasan) dan `approval_requests`/`approval_decisions` (multi-penyetuju, `policy_snapshot`)
  - `setting_audit_logs` kini mencatat juga kebijakan persetujuan dan keputusannya
  - `setting_definitions.requires_approval` hanya default seed; kolom baru `approval_floor`
  - guardrail G1–G7 dan seed default policies
- **PRD:** matriks §17.3 ditambah kolom *Persetujuan default*; §17.5 permission diperbarui; §18 baru (Kebijakan Persetujuan, tab Persetujuan, Kotak Persetujuan, default, pengaman, acceptance criteria).
- **TRD:**
  - §15.5 langkah 6 kini memanggil resolver persetujuan
  - §16 baru: `resolveApprovalPolicy`, alur request/decide, meta-approval, floor, validasi anti-kebuntuan, snapshot, API, test
- **AGENTS:** prinsip persetujuan lewat resolver; tugas modul approval.
- **CLAUDE:** aturan #12a (persetujuan dinamis, tanpa bypass, tanpa self-approval, item terkunci).
- **Default:** On (Owner, 1 penyetuju, floor) untuk BPJS/PPh21, bonus, gaji gantungan, tarif, dan peran & akses. Meta-approval = Owner + HR Admin (2). Lainnya Off (audit saja). `system.public_api` tetap terkunci mati.
- Prototipe tidak diubah, dan tidak ada data pribadi yang dimasukkan.
