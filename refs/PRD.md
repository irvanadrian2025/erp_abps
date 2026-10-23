# Product Requirements Document
## Sistem HRD — Absensi & Payroll · Ayam Bakar Pucuk Singkong (ABPS)
**Versi:** 2.2 (v2.1 + aturan persetujuan dinamis)
**Tanggal:** 24 Oktober 2026
**Status:** Draft — menunggu konfirmasi pertanyaan terbuka (lihat §16 & `CHANGELOG-analisa.md`)
**Prototipe:** https://digihrd.vercel.app/

---

## 1. Ringkasan Produk

ABPS saat ini mengelola presensi dan penggajian **secara manual** (Google Sheet/Excel per bulan & per minggu, jadwal shift di sheet, pengumuman via grup WhatsApp, slip gaji dari template sheet). Sistem ini mendigitalisasi seluruh alur: **jadwal shift → presensi GPS → izin/sakit/lembur → kasbon/borongan/bonus → payroll mingguan & bulanan → slip gaji → laporan**, dengan aturan hitung yang **sama persis** dengan rumus sheet yang berlaku sekarang.

Unit kerja yang dicakup:

| Unit | Karyawan (Okt 2026) | Frekuensi gaji | Basis |
|---|---|---|---|
| **Outlet Baros** (crew) | ±12: Manager, Leader, Kasir ×2, Kasir&Server, Checker ×2, Server ×2, Kitchen ×2, Kitchen&Server | Bulanan | Gaji/bulan, pecah Pokok / Tunj. Tidak Tetap (/ Tunj. Jabatan), basis 26 hari |
| **Dapur Pusat – Produksi** | 4 tetap (P001–P004) + 1 freelance (P005) | **Mingguan, Sabtu–Jumat** | Gaji harian ÷ 9 = upah per jam, dihitung per menit |
| **Kantor Pusat** | Admin, Manager Operasional, Purchasing | Bulanan | Pokok / Tunj. (/ Jabatan), basis 26 hari |
| **Part-time** | Content Specialist | Bulanan | Fixed + per kunjungan |

## 2. Tujuan & Sasaran

### 2.1 Tujuan Bisnis
- Menghapus rekap manual jam masuk/pulang & rumus sheet yang rawan salah (ditemukan `#REF!` di template "Payslip Kantor", dan rumus Total Jam yang tidak konsisten antar sheet).
- Payroll mingguan pusat dan bulanan outlet/kantor dihasilkan dalam < 10 menit per periode, **selisih Rp0** terhadap hitungan sheet pada data uji (paralel run minimal 1 bulan / 4 minggu).
- Transparansi potongan (telat, izin, sakit, kasbon) dan bonus omset bagi karyawan.
- Kehadiran tervalidasi lokasi (geofence Outlet Baros & Dapur Pusat).

### 2.2 Sasaran Pengguna
| Peran | Contoh di ABPS | Kebutuhan utama |
|---|---|---|
| HR Admin | Admin pusat | Generate payroll, kasbon, slip, laporan |
| Manager | Manager Operasional | Jadwal, approval izin, input omset & bonus |
| Supervisor | Leader outlet | Konfirmasi tukar shift, input borongan, checklist |
| Employee | Crew, produksi, part-time | Clock-in/out HP, lihat jadwal, ajukan izin/sakit, lihat slip |

## 3. Ruang Lingkup

### 3.1 In-Scope (v2.0)
- Presensi GPS + foto (opsional) dengan geofence per lokasi.
- Jadwal shift outlet (P1/P2/Middle/Shift 3) + station (ck/kit/ks/se) + status off/off izin/off sakit; jadwal produksi pusat.
- Izin, sakit (lampiran surat menyusul H+1), tukar shift (wajib backup + konfirmasi Leader).
- Lembur (aturan pusat otomatis; outlet/kantor via pengajuan jam).
- Payroll **mingguan** (pusat) & **bulanan** (outlet, kantor, part-time) dengan komponen ABPS.
- Kasbon, borongan/kurir/metik, insentif (masuk < 05:00, tanggal merah), bonus omset outlet & catering, bonus akhir tahun, gaji gantungan.
- Slip gaji PDF format ABPS; export Excel rekap (format mirip sheet lama) & rekap transfer bank.
- Import data awal dari Excel (karyawan, gaji).

### 3.2 Out-of-Scope (v2.0)
- Integrasi otomatis POS (MOKA/Runchise) & akuntansi (Kledo) — omset diinput manual dulu (Q-11).
- Kas outlet, stock opname, rekap masalah/komplain (STARS) — tetap di sheet; hanya *checklist harian* yang opsional dimasukkan (fase 2).
- Pembayaran/transfer bank otomatis.
- BPJS & PPh21 **non-aktif** (komponen tersedia, dimatikan) sampai dikonfirmasi (Q-06).
- Cuti tahunan berbasis saldo (tidak ditemukan di data lapangan).

## 4. Definisi Peran & Hak Akses

### 4.1 Hierarki Peran

```
SUPER_ADMIN (Vendor / pengembang)
    └── TENANT_ADMIN (Perusahaan)
            ├── HR_ADMIN
            ├── HR_STAFF
            ├── MANAGER
            ├── SUPERVISOR
            └── EMPLOYEE
```

### 4.2 Matriks Akses Lengkap

| Fitur / Modul | HR_ADMIN | HR_STAFF | MANAGER | SUPERVISOR | EMPLOYEE |
|---|:---:|:---:|:---:|:---:|:---:|
| **Manajemen Karyawan** |||||
| Tambah/Edit/Hapus karyawan | ✅ | ✅ | ❌ | ❌ | ❌ |
| Lihat profil semua karyawan | ✅ | ✅ | ✅ (tim) | ✅ (tim) | ❌ |
| Lihat profil sendiri | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Absensi** |||||
| Clock-in / Clock-out | ❌* | ❌* | ✅ | ✅ | ✅ |
| Lihat rekap absensi semua | ✅ | ✅ | ✅ (tim) | ✅ (tim) | ❌ |
| Lihat absensi sendiri | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit / Koreksi absensi | ✅ | ✅ | ❌ | ❌ | ❌ |
| Approval izin/cuti | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Lokasi Absensi** |||||
| Tambah/Edit lokasi & radius | ✅ | ❌ | ❌ | ❌ | ❌ |
| Lihat peta lokasi | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Jadwal Shift** |||||
| Buat/Edit template shift | ✅ | ✅ | ❌ | ❌ | ❌ |
| Assign shift ke karyawan | ✅ | ✅ | ✅ | ✅ | ❌ |
| Lihat jadwal sendiri | ✅ | ✅ | ✅ | ✅ | ✅ |
| Swap shift (request) | ❌ | ❌ | ❌ | ❌ | ✅ |
| Approve swap shift | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Payroll** |||||
| Generate payroll run | ✅ | ❌ | ❌ | ❌ | ❌ |
| Review/Approve payroll | ✅ | ✅ | ❌ | ❌ | ❌ |
| Lihat semua slip gaji | ✅ | ✅ | ❌ | ❌ | ❌ |
| Cetak slip gaji karyawan | ✅ | ✅ | ❌ | ❌ | ❌ |
| Lihat & cetak slip sendiri | ✅ | ✅ | ✅ | ✅ | ✅ |
| Atur komponen gaji | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Laporan** |||||
| Laporan kehadiran (semua) | ✅ | ✅ | ✅ (tim) | ✅ (tim) | ❌ |
| Laporan payroll (semua) | ✅ | ✅ | ❌ | ❌ | ❌ |
| Export CSV/Excel | ✅ | ✅ | ✅ (tim) | ❌ | ❌ |
| **Pengaturan Sistem** |||||
| Konfigurasi tenant (logo, warna) | ✅ | ❌ | ❌ | ❌ | ❌ |
| Atur rules keterlambatan | ✅ | ❌ | ❌ | ❌ | ❌ |
| Atur komponen BPJS/Pajak (opsional, non-aktif ABPS) | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Khusus ABPS (v2.0)** |||||
| Input kasbon | ✅ | ✅ | ❌ | ❌ | ❌ |
| Input borongan/kurir/metik | ✅ | ✅ | ✅ | ✅ (Leader) | ❌ |
| Input omset outlet & bonus | ✅ | ❌ | ✅ (Manager Ops) | ❌ | ❌ |
| Konfirmasi tukar shift | ✅ | ✅ | ✅ | ✅ (Leader) | ❌ |
| Isi checklist harian station | ❌ | ❌ | ✅ | ✅ | ✅ |

> *HR_ADMIN dan HR_STAFF tidak melakukan absensi lapangan, mereka dikecualikan dari kewajiban clock-in/out reguler. Dapat dikonfigurasi per tenant.

---

**Pemetaan peran ABPS (usulan, perlu konfirmasi Q-13):** HR_ADMIN = Admin pusat (Yuni) / owner; MANAGER = Manager Operasional (Dahlia); SUPERVISOR = Leader outlet (Nasir); EMPLOYEE = crew outlet, produksi pusat, part-time. Owner ("Abah") dapat diberi HR_ADMIN read-only + approve.
---

## 5. Fitur: Presensi Geolokasi

### 5.1 Alur Clock-In / Clock-Out
1. Karyawan login (nomor karyawan + PIN/password) di HP → halaman **Presensi**.
2. Sistem mengambil GPS (akurasi ≤ 50 m) + deteksi mock GPS + foto selfie (jika lokasi `require_photo`).
3. Dalam radius → status `valid`. Di luar radius → modal alasan (jika lokasi mengizinkan) → status `outside_radius`, perlu approval.
4. Jam masuk dibandingkan dengan shift hari itu (dari jadwal) → `late_minutes`.
5. Clock-out → hitung durasi kerja, kandidat lembur (lihat §8).
6. Lupa clock-out → notifikasi ke karyawan + Leader; koreksi oleh HR dengan audit log.

### 5.2 Lokasi
| Lokasi | Unit | Radius awal | Catatan |
|---|---|---|---|
| Outlet Baros | Crew outlet | 100 m | Koordinat belum ada (Q-09) |
| Dapur Pusat | Produksi, kantor, freelance | 100 m | Koordinat belum ada; apakah kantor = dapur pusat? (Q-09) |

Kurir freelance bekerja di luar lokasi → presensi tidak wajib; pekerjaan dicatat sebagai **entri borongan** (§8.6).

### 5.3 Modal yang Muncul
Tetap seperti v1.0: *GPS tidak aktif*, *di luar radius (tampilkan jarak)*, *mock GPS terdeteksi (blokir + flag)*, *sudah clock-in*, *tidak ada jadwal hari ini (tanya: masuk sebagai tambahan?)*.

### 5.4 Deteksi GPS Palsu
Sama dengan v1.0 (flag `is_mock_gps`, notifikasi HR Admin).

---

## 6. Fitur: Jadwal Shift

### 6.1 Template Shift Outlet (jadwal Oktober 2026)
| Kode | Nama | Jam | Keterangan |
|---|---|---|---|
| P1 | Shift 1 / Pagi | 06.00–16.00 | |
| P2 | Shift 2 / Siang | 11.30–21.30 | |
| MID | Middle | 09.30–19.30 | "lembur jika dibutuhkan" |
| S3 | Shift 3 | 12.30–21.30 | varian 11.30–21.30 juga muncul |
| — | Custom | bebas | contoh 09.00–19.00, 10.00–20.00 (sistem harus mengizinkan jam custom per hari) |

Outlet buka 07.00–21.00 (closing 30 menit, take away s/d 20.30). Versi jadwal lama: P1 08.00–20.00, P2 09.30–21.30, istirahat 30 menit. Shift diganti → buat template baru, jangan ubah template lama (riwayat).

### 6.2 Station & Status Harian
- Station per hari: `ck` Checker, `kit`/`kc` Kitchen, `ks` Kasir, `se` Server (+ `bk` Bakar dari jobdesk). Rotasi station per shift.
- Status: `work`, `off`, `off_izin`, `off_sakit`.
- Tampilan grid bulanan seperti sheet (baris = karyawan, kolom = tanggal 1–31), bukan hanya grid mingguan Senin–Minggu seperti prototipe saat ini.
- Penanda hari khusus: **payday** (tgl 3 pada jadwal Okt 2026), tanggal merah.

### 6.3 Produksi Pusat
Tidak memakai shift outlet. Jam hitung mulai **05.00** (masuk lebih awal tidak dihitung, tapi dapat insentif). Hari kerja Sabtu–Jumat; hari Minggu kadang dibayar terpisah (Q-03).

### 6.4 Tukar Shift & Izin (aturan jadwal)
- Tukar shift **wajib** ada pengganti (backup) dan dikonfirmasi Leader → alur: pemohon pilih pengganti → pengganti setuju → Leader approve.
- Izin/sakit wajib diajukan **paling lambat malam sebelumnya atau 6 jam sebelum shift**; pengajuan terlambat ditandai `late_notice` (sanksi belum didefinisikan, Q-10).
- Surat sakit boleh diunggah **H+1**.

---

## 7. Fitur: Aturan Keterlambatan

Diganti dari "rules per grade" generik menjadi **aturan per pay group, berversi tanggal** (`pay_rules`), karena aturan lapangan berubah dari waktu ke waktu:

| Pay group | Potongan telat | Sumber |
|---|---|---|
| Crew outlet | **Rp500 per menit** sejak menit pertama (toleransi 0) | Gaji Bulanan Sept/Okt 2026 |
| Kantor pusat | **prorata**: jam telat × (gaji ÷ 26 ÷ 9), dibulatkan | baris Admin, Sept 2026 |
| Produksi pusat | Tidak ada potongan; upah memang dihitung per menit dari jam masuk (setelah 05.00) | Gaji Mingguan |

Prototipe saat ini menyediakan metode `per_minute`, `per_hour`, `flat_cut`, `none` — dipertahankan, ditambah `prorata_hourly`.

---

## 8. Fitur: Payroll ABPS

### 8.1 Pay Group & Periode
| Pay group | Frekuensi | Periode | Payday |
|---|---|---|---|
| WEEKLY_PUSAT | Mingguan | **Sabtu–Jumat** | Jumat/akhir periode (konfirmasi Q-07) |
| MONTHLY_OUTLET | Bulanan | tgl 1–akhir bulan (asumsi) | tgl 3 bulan berikutnya (jadwal Okt "3/payday") |
| MONTHLY_KANTOR | Bulanan | sama | sama |
| PART_TIME | Bulanan | sama | sama |

Satu bulan bisa memiliki 4–5 payroll run mingguan. Run dibuat per (pay group, periode). Status: `draft → calculated → reviewed → approved → paid` (sama dengan prototipe).

### 8.2 Produksi Pusat (mingguan)
Per hari kerja:
- **Jam hitung masuk** = MAX(jam masuk, 05:00). Masuk < 05:00 → **Insentif Rp10.000**.
- **Total jam** = jam pulang − jam hitung masuk (Q-04: sheet Okt sebagian memakai jam masuk asli).
- **Upah hari** = total jam (menit/60) × upah per jam; **upah per jam = gaji harian ÷ 9** (Lilis 130rb, Ade 120rb, Yuyun 100rb, Leni 90rb). Sheet Des–Jan memakai tarif jam langsung (12,5rb/11rb/9,5rb/8,5rb) → sistem menyimpan `daily_rate` & riwayat.
- **Hari kerja full 9 jam** (kolom "Hari Kerja Full 9 Jam"): sejak ±Agustus, hari tertentu (Jumat) dibayar **flat = gaji harian** (Q-05).
- **Lembur** = jika total ≤ 9 jam → 0; jika kelebihan ≤ 30 menit → 0; selain itu **ROUNDUP(kelebihan − 30 menit, jam)** × **Rp10.000/jam**.
- **Insentif tanggal merah** Rp50.000 bila masuk di tanggal merah.
- **Penambahan lain**: bonus akhir tahun (mis. 27 Des–2 Jan), hari Minggu/jam tambahan dibayar terpisah.
- **Gaji bersih** = Upah + Lembur + Insentif + Penambahan − **Kasbon**. Dibayar transfer (BCA).

### 8.3 Freelance Pusat (P005)
- Upah jam **Rp9.000/jam** + borongan **metik cengek Rp2.500/kg**, **kupas baput Rp3.000/kg** + **kurir Rp20.000/tugas** (antar bahan, buang sampah TPS, antar pesanan). Rekap bulanan.

### 8.4 Crew Outlet & Kantor (bulanan)
- **Gaji/bulan** dipecah menjadi komponen (nominal per karyawan, effective-dated):
  - Crew: 50% Gaji Pokok + 50% Tunjangan Tidak Tetap (kehadiran & transport).
  - Leader: 50% / 30% / 20% (Pokok / Tunj. Tidak Tetap / Tunj. Jabatan).
  - Admin: 60/40; Manager Operasional: 60/25/15.
- **Gaji per hari** = gaji/bulan ÷ **26**.
- **Kenaikan pertengahan bulan / karyawan baru / percobaan**: prorata per hari kerja (÷26). Masa percobaan 1,8jt → 2jt setelah 3 bulan (rencana).
- **Lembur**: jam lembur (boleh 0,5 jam) × **Rp10.000** (crew) / **Rp17.000** (kantor).
- **Potongan**:
  - Telat: lihat §7.
  - **Izin** — crew: hari × (gaji total ÷ 26); kantor: hari × (tunj. tidak tetap ÷ 26).
  - **Sakit** — crew: hari × (tunj. tidak tetap ÷ 26); kantor: Rp50.000/hari.
  - **Absent prorate** (tidak masuk / belum bergabung).
  - **Kasbon** (dicatat di modul kasbon, dipotong otomatis).
- **Tambahan**: bonus omset outlet, bonus catering, transport, insentif tanggal merah Rp50.000/hari.
- **Libur operasional** (Lebaran 18–28 Mar 2026): dibayar **50%**.
- **Gaji gantungan**: sebagian gaji ditahan (Maret: Rp450.000 × 2 minggu) dan dilepas kemudian — tujuan & aturan pelepasan perlu konfirmasi (Q-14).
- **THP** = total pendapatan − total potongan.

> v2.1: semua angka & metode di §7–§9 adalah default; lihat §17.3. Semua angka di atas **dikonfigurasi** di `pay_rules` dan komponen karyawan — riwayat menunjukkan aturan berubah (Maret 2026: izin Rp80.000/hari, sakit Rp50.000/hari).

### 8.5 Part-time
Fixed bulanan + fee per kunjungan (Content Specialist: Rp1.200.000 + Rp50.000 × jumlah visit, contoh 9 visit).

### 8.6 Kasbon & Borongan
- **Kasbon**: tanggal, nominal, sumber (kas outlet / kas besar / "Abah"), cicilan opsional; otomatis jadi potongan pada run berikutnya; saldo tampil di slip.
- **Borongan/kurir**: Leader/Admin mencatat per hari: item, qty, tarif snapshot.

### 8.7 Bonus Omset Outlet
Dari pengumuman "Bonus Outlet" (WA, 8 Jan) — kondisi **12 karyawan**:
| Omset bersih / bulan | Bonus |
|---|---|
| < Rp350 jt | Tidak ada |
| Rp350–400 jt | 0,5% × total omset bersih |
| > Rp400 jt | 1% × total omset bersih |

**Omset bersih** = omset kotor − diskon − potongan aplikasi ojol. Manager input omset bulanan → sistem hitung *pool* → alokasi per karyawan (manual / rata / bobot kehadiran). Data Sept 2026: mayoritas 180rb, yang izin 80rb; total 1,76 jt = 0,5% × 352 jt (**inferensi, Q-02**). Bonus catering 0,5% (Admin) — Q-15.

### 8.8 Alur Proses Payroll
1. HR memilih pay group + periode → **Generate** (draft).
2. Sistem menarik: presensi, jadwal, izin/sakit, lembur approved, kasbon open, borongan, visit, bonus allocations, holidays.
3. Kalkulasi → `calculated`; HR koreksi manual dengan catatan (audit log).
4. Review (Manager/HR) → Approve (owner/HR Admin) → slip terbit → tandai `paid` setelah transfer.
5. Paralel run dengan sheet selama fase transisi; laporan selisih per karyawan.

---

## 9. Fitur: Slip Gaji

### 9.1 Slip Bulanan (format template "SLIP GAJI KARYAWAN AYAM BAKAR PUCUK SINGKONG")
Header: logo, judul, periode, nama, jabatan, unit. Penandatangan: **HR PUCUK SINGKONG**.

| Pendapatan | Potongan |
|---|---|
| Gaji Pokok | Kasbon |
| Tunjangan Tidak Tetap | Telat (… menit) |
| Tunjangan Jabatan | Izin (… hari) |
| Lembur (… jam) | Absent Prorate |
| Bonus | Sakit (… hari) |
| Transport | Gaji Gantungan (jika ada) |
| **Total** | **Total** |
| **THP (Take Home Pay)** | |

### 9.2 Slip Mingguan (Produksi Pusat)
Tabel harian Sabtu–Jumat: tanggal, jam masuk, jam hitung, jam pulang, total jam, upah, lembur, insentif; lalu Penambahan, Kasbon, **Gaji Bersih**.

### 9.3 Format & Akses
PDF (A5/A4), dapat diunduh karyawan sendiri; batch PDF/ZIP untuk HR; tanpa nomor rekening/NIK di slip.

---

## 10. Konfigurasi Tenant (Branding)
Nama: **Ayam Bakar Pucuk Singkong** (prototipe: "PT Pucuk Singkong Sejahtera" — Q-12). Warna utama **#1F8802**, sidebar/aksen **#A76A02**, font **Cabin / Albert Sans** (sesuai `/settings/app` prototipe). Setting lain sama dengan v1.0 (logo, favicon, timezone WIB, IDR).

## 11. Notifikasi
| Trigger | Penerima | Channel |
|---|---|---|
| Jadwal bulan baru terbit | Crew | In-app (+ WA manual, fase 2 otomatis) |
| Pengajuan izin/sakit < 6 jam sebelum shift | Leader | In-app |
| Permintaan tukar shift | Pengganti → Leader | In-app |
| Lupa clock-out | Karyawan + Leader | In-app |
| Payroll run siap review | Manager/HR | In-app + email |
| Slip tersedia | Karyawan | In-app |
| GPS palsu | HR Admin | In-app + email |

## 12. Laporan & Dashboard
- Dashboard: hadir/telat/off hari ini per lokasi; status run mingguan & bulanan; total kasbon outstanding; progres omset vs target bonus.
- Laporan: rekap kehadiran, keterlambatan (menit & rupiah), lembur, payroll per pay group (format kolom seperti sheet lama), rekap transfer, kasbon, borongan, bonus.
- Export: Excel (.xlsx), PDF. Kolom BPJS/PPh21 disembunyikan bila non-aktif.

## 13. User Stories Utama
| ID | Story |
|---|---|
| US-01 | Sebagai crew, saya clock-in dari HP di Outlet Baros dan langsung tahu apakah saya telat. |
| US-02 | Sebagai crew, saya melihat jadwal bulan ini (shift + station + off). |
| US-03 | Sebagai crew, saya mengajukan tukar shift dengan memilih pengganti, lalu Leader mengonfirmasi. |
| US-04 | Sebagai crew, saya mengajukan sakit dan mengunggah surat keesokan harinya. |
| US-05 | Sebagai karyawan produksi, saya melihat upah mingguan per hari, lembur, insentif, dan kasbon. |
| US-06 | Sebagai karyawan, saya mengunduh slip dan memahami setiap potongan. |
| US-10 | Sebagai Admin, saya generate payroll mingguan pusat tiap Jumat dan bulanan outlet tiap awal bulan. |
| US-11 | Sebagai Admin, saya mencatat kasbon dan sistem memotong otomatis. |
| US-12 | Sebagai Manager Ops, saya input omset bersih bulanan dan sistem menghitung pool bonus sesuai tier. |
| US-13 | Sebagai Leader, saya mencatat tugas kurir & hasil metik per hari. |
| US-14 | Sebagai Admin, saya mengubah tarif/aturan dengan tanggal berlaku tanpa merusak hitungan periode lama. |

## 14. Acceptance Criteria Utama
| Fitur | Kriteria |
|---|---|
| Presensi | Tersimpan dengan koordinat, akurasi, status; tidak bisa clock-in dua kali sehari |
| Payroll mingguan | Data uji minggu 10–16 Okt 2026 (P001–P004) menghasilkan angka = sheet (selisih Rp0, setelah Q-04 diputuskan) |
| Payroll bulanan | Data uji Sept 2026 outlet & kantor = sheet (selisih Rp0) termasuk telat/izin/sakit/lembur/bonus/kasbon |
| Aturan berversi | Mengubah tarif tidak mengubah run yang sudah `approved` |
| Slip | Kolom sesuai template ABPS; generate ≤ 3 detik |
| Keamanan | **Semua** endpoint `/api/*` menolak tanpa sesi (prototipe saat ini terbuka publik — wajib diperbaiki); data pribadi tidak tampil ke role yang tidak berhak |

## 15. Asumsi & Keterbatasan
- Basis 26 hari berlaku untuk semua karyawan bulanan.
- Periode bulanan = bulan kalender.
- Satu karyawan satu lokasi utama; perbantuan antar lokasi dicatat sebagai presensi di lokasi lain.
- Omset diinput manual; sistem tidak menarik data POS.
- Koneksi internet HP karyawan tersedia di lokasi.

## 16. Keputusan Konfigurasi
Mulai v2.1 tidak ada pertanyaan yang menghambat pengembangan. Setiap Q-xx/A-x menjadi pengaturan dinamis (§17). Angka dan pilihan di §5–§12 adalah **default awal** dari sheet, bukan aturan permanen. Tabel lengkapnya ada di `CHANGELOG-analisa.md` §5.

---

## 17. Pengaturan Aturan Dinamis (v2.1)

Semua pertanyaan Q-01…Q-17 dan asumsi A1–A7 **tidak diputuskan sebagai satu pilihan**. Masing-masing menjadi pengaturan yang bisa di-on/off-kan atau diganti metodenya. Angkanya bisa diubah kapan saja oleh admin, berlaku per tanggal (effective_from/to), dan bisa dibuat khusus per kelompok gaji, lokasi, jabatan, atau karyawan. Setiap perubahan tercatat di audit log. Default awal = praktik di sheet saat ini, jadi hasil hitung awal tetap sama dengan sheet.

### 17.1 Layar "Pengaturan Aturan" (`/settings/rules`)
- **Navigasi per kategori:** Fitur (toggle), Presensi & Telat, Lembur, Potongan, Mingguan Pusat, Periode & Payday, Bonus & Omset, Kasbon & Borongan, Slip, Peran & Akses.
- **Kartu per setting:**
  - label dan penjelasan
  - nilai efektif saat ini beserta sumbernya ("Default", "Kelompok: WEEKLY_PUSAT", "Karyawan: P003")
  - badge Q-xx
  - kontrol sesuai tipe: switch, dropdown opsi, input angka/jam/persen, editor tabel tier
- **Tab Override:** daftar override per kelompok gaji, lokasi, jabatan, dan karyawan. Tombol "Tambah override" memilih scope dan tanggal berlaku.
- **Tab Riwayat:** timeline versi (berlaku dari–sampai, nilai, pembuat, penyetuju, alasan).
- **Form ubah:**
  - wajib diisi: nilai baru, tanggal berlaku mulai, opsional tanggal berakhir, alasan
  - tombol **Preview dampak**: simulasi selisih per karyawan pada run yang belum di-approve
  - jika aturan persetujuan setting itu aktif (§18, dinamis), perubahan masuk *Menunggu persetujuan*
- **Tab Persetujuan** per kartu (v2.2): status persetujuan, penyetuju, dan riwayat; hanya Owner yang dapat mengubah (§18)
- **Penjelasan per karyawan** (`/employees/:id/rules`): semua nilai efektif untuk karyawan itu pada tanggal tertentu, lengkap dengan sumbernya.
- **Audit log** (`/settings/audit`): filter per key, tanggal, dan pengguna; bisa diekspor ke Excel.
- **Proteksi:**
  - Perubahan yang menyentuh periode payroll yang sudah approved/paid ditolak, kecuali melalui *Run Penyesuaian*.
  - `system.public_api` terkunci dan tidak bisa diubah dari layar.

### 17.2 Urutan Override
Karyawan > Jabatan > Lokasi > Kelompok gaji > Tenant > Default sistem. Pada level yang sama, versi yang berlaku ditentukan oleh tanggal acuan: akhir periode payroll, atau tanggal presensi untuk aturan harian.

### 17.3 Matriks Q/A → Pengaturan → Opsi → Default
| Q/A | Setting key | Hal | Tipe | Opsi yang tersedia | Default awal (dari sheet) | Scope override | Siapa ubah | Persetujuan default (v2.2, dinamis) |
|---|---|---|---|---|---|---|---|---|
| Q-01 | `bonus.outlet.tiers` | Tier bonus outlet (batas & persen) | json tiers | daftar tier bebas {min,max,percent}; boundary per tier: `inclusive`/`exclusive`; headcount_rule: `fixed`/`scale_by_headcount`/`ignore` | <350jt 0% · 350–400jt 0,5% · >400jt 1%; 400jt = inklusif ke 0,5%; headcount ref 12 = `ignore` | location | HR Admin + Owner | On · Owner · floor |
| Q-02 | `bonus.outlet.allocation_method` | Pembagian pool bonus | enum | `equal` (rata) · `attendance_days` (proporsional hari hadir) · `position_weight` (bobot per jabatan) · `attendance_x_weight` · `manual` · `fixed_amount_with_penalty` (nominal tetap, dikurangi jika izin/sakit) | `manual` (praktik Sept: 180rb/80rb); `bonus.outlet.eligible_positions` = semua crew + Manager Ops | location | HR Admin + Owner | On · Owner · floor |
| Q-03 | `weekly.sunday_policy` | Kerja hari Minggu pusat | enum + angka | `normal_hourly` · `separate_hourly_rate` (tarif sendiri) · `multiplier` · `flat_per_day` · `excluded` | `normal_hourly`, dibayar di baris terpisah "Gaji di Hari Minggu/Jam" | pay_group/employee | HR Admin | Off (audit saja) |
| Q-04 | `weekly.total_hours_basis` | Dasar Total Jam pusat | enum | `counted_clock_in` (mulai 05:00) · `raw_clock_in` (jam masuk asli) | `counted_clock_in`; `weekly.earliest_count_time` = 05:00 | pay_group/employee | HR Admin | Off (audit saja) |
| Q-05 | `weekly.full_day_flat` | Hari "Full 9 Jam" dibayar flat | toggle + rule | on/off; `days_of_week` (multi), `min_minutes`, `pay`: `daily_rate`/`hourly` | on; Jumat; min 0 menit; `daily_rate` | pay_group/employee | HR Admin | Off (audit saja) |
| Q-06 | `feature.bpjs / feature.pph21` | BPJS Kes/TK & PPh21 | toggle + angka | on/off masing-masing; persen & plafon per program; metode PPh21 `TER`/`none`; ditanggung `employee`/`company` | off / off | tenant/pay_group/employee | HR Admin | On · Owner · floor |
| Q-07 | `period.monthly_cutoff · payday` | Periode & payday | angka/enum | cutoff 1–28 atau `calendar`; payday tgl N / `last_working_day`; mingguan `week_start_dow` 0–6, payday dow | bulanan `calendar`, payday tgl 3; mingguan Sabtu–Jumat, payday Jumat | pay_group | HR Admin | Off (audit saja) |
| Q-08 | `rate.* (daily_rate, hourly_rate, salary components)` | Tarif berlaku (saat ini vs planing) | angka berversi | nilai per karyawan/jabatan dengan effective_from; rencana disimpan sebagai versi `scheduled` (tanggal masa depan) | tarif "saat ini"/slip Sept–Okt; "planing" = draft belum aktif | employee/position | HR Admin + Owner | On · Owner · floor |
| Q-09 | `attendance.geofence` | Lokasi, radius, foto, kantor=dapur | angka/toggle | lat/lng, radius m, `require_photo` on/off, `allow_outside_with_reason` on/off, `mock_gps_action`: `block`/`flag`; karyawan boleh multi-lokasi on/off | radius 100 m, foto on, luar radius perlu alasan, mock = block; koordinat kosong → clock-in GPS non-aktif sampai diisi | location/employee | HR Admin | Off (audit saja) |
| Q-10 | `attendance.late_tolerance · leave.notice_rule` | Toleransi telat & izin mendadak | angka/enum | toleransi menit; `late_deduction_mode`: `per_minute`/`prorata_hourly`/`per_hour`/`flat_cut`/`tiered`/`none`; batas notice jam; sanksi notice: `none`/`flag`/`flat`/`as_alpha`; surat sakit H+N | crew 0 mnt, 500/mnt; kantor prorata; notice 6 jam/malam sebelum → `flag`; surat sakit H+1 | pay_group/position/employee | HR Admin | Off (audit saja) |
| Q-11 | `revenue.source` | Sumber omset bersih | enum multi | `manual` · `import_moka` · `import_runchise` · `import_kledo` · `import_ojol` (CSV/XLSX); `net_formula` komponen pengurang (diskon, ojol, pajak, retur) on/off; `require_verification` on/off | `manual`; pengurang = diskon + potongan ojol; verifikasi on (Owner) | location | Manager Ops input, Owner verifikasi | Off (audit saja) |
| Q-12 | `tenant.display_name · slip.signatory` | Nama di aplikasi/slip | teks | nama aplikasi, nama di slip, penandatangan, logo | app "Ayam Bakar Pucuk Singkong"; slip "SLIP GAJI KARYAWAN AYAM BAKAR PUCUK SINGKONG"; ttd "HR PUCUK SINGKONG" | tenant | HR Admin | Off (audit saja) |
| Q-13 | `rbac.role_permissions · employee.system_user` | Peran & siapa masuk sistem | matriks | role → permission (bisa custom role); per karyawan `has_login` on/off, `in_payroll` on/off | default matriks PRD §4; Manager Produksi & kurir: in_payroll on, has_login off | tenant/employee | HR Admin + Owner | On · Owner · floor |
| Q-14 | `feature.salary_holdback` | Gaji gantungan | toggle + rule | on/off; nominal `fixed`/`percent`; frekuensi per run; `release_rule`: `manual`/`after_n_runs`/`on_resign`/`date`; berlaku siapa (pay_group/employee) | off (riwayat Maret 450rb×2 bisa dicatat manual) | pay_group/employee | HR Admin + Owner | On · Owner · floor |
| Q-15 | `feature.bonus_catering` | Bonus catering | toggle + angka | on/off; persen; basis omset `catering_net`/`catering_gross`/`manual`; penerima (daftar karyawan/jabatan); alokasi seperti Q-02 | on, 0,5%, basis `manual`, penerima Admin (asumsi A6) | tenant/employee | HR Admin + Owner | On · Owner · floor |
| Q-16 | `overtime.*` | Lembur outlet/kantor/pusat | enum + angka | `mode`: `flat_per_hour`/`multiplier`/`tiered`; tarif; `threshold_minutes`; `grace_minutes`; `rounding`: `none`/`ceil_hour`/`floor_half`/`nearest_half`/`ceil_half`; `source`: `auto_from_attendance`/`request_approval`/`both`; approver role; min durasi | pusat: flat 10rb, 540 mnt, grace 30, `ceil_hour`, auto. Crew: flat 10rb, `nearest_half`, `request_approval` (Leader). Kantor: flat 17rb, `request_approval` (Manager) | pay_group/position/employee | HR Admin | Off; tarif flat: On · Owner · floor |
| Q-17 | `system.demo_data · system.public_api` | Data dummy & API publik prototipe | toggle | `demo_data` on/off (hapus seed); `public_api` selalu off di production (tidak bisa diubah via UI) | demo off saat go-live; public_api off (hard rule, bukan setting) | tenant | Developer/Super Admin | 🔒 terkunci mati |
| A1 | `period.monthly_cutoff` | Periode bulanan & payday | — | lihat Q-07 | calendar, tgl 3 | pay_group | HR Admin | Off (audit saja) |
| A2 | `location.mapping` | Kantor = Dapur Pusat | enum | `same_location`/`separate_location` | `same_location` | tenant | HR Admin | Off (audit saja) |
| A3 | `shift.production_template` | Jam produksi pusat | angka | start/end/break per template, berversi | 05:00–14:00, break 0 | pay_group | HR Admin | Off (audit saja) |
| A4 | `attendance.late_tolerance` | Toleransi telat crew | angka | lihat Q-10 | 0 menit | pay_group | HR Admin | Off (audit saja) |
| A5 | `bonus.outlet.allocation_method` | Pool bonus Sept | — | lihat Q-01/Q-02 | manual; pool dihitung dari omset input | location | HR Admin + Owner | On · Owner · floor |
| A6 | `feature.bonus_catering` | Bonus Admin = catering | — | lihat Q-15 | on untuk Admin | employee | HR Admin + Owner | On · Owner · floor |
| A7 | `rate.*` | Tarif saat ini vs planing | — | lihat Q-08 | saat ini | employee | HR Admin + Owner | On · Owner · floor |

### 17.4 Aturan Bonus Outlet (tetap dicakup, kini dinamis)
- Default tier dari pengumuman WA 8 Jan:
  - < 350 jt: 0%
  - 350–400 jt: 0,5%
  - > 400 jt: 1%
  Semua dari omset bersih (kotor − diskon − potongan ojol), dengan referensi 12 karyawan.
- Semuanya bisa diubah dan berversi tanggal: tier, persen, sifat batas (inklusif/eksklusif), komponen pengurang omset, aturan headcount, metode pembagian (rata / hari hadir / bobot jabatan / hadir × bobot / manual / nominal tetap dengan penalti), dan daftar jabatan yang berhak.
- Pool dan alokasi selalu bisa di-*override manual* per karyawan, dengan alasan yang masuk audit log.

### 17.5 Siapa Mengubah
| Permission | Default role |
|---|---|
| `settings.view` | HR Admin, HR Staff, Manager |
| `settings.edit` | HR Admin |
| `settings.approve` | Sesuai `approval_policies` per setting (default Owner); pembuat ≠ penyetuju |
| `settings.approval_policy.manage` | Owner (perubahan disetujui Owner + HR Admin, §18) |
| `revenue.input` / `revenue.verify` | Manager Ops / Owner |
| `payroll.adjust` | HR Admin + persetujuan Owner |
Permission ini juga dapat dikonfigurasi (Q-13, `custom_roles`).

### 17.6 Acceptance Criteria Tambahan
| Kriteria | Target |
|---|---|
| Ubah setting tanpa deploy | Berlaku pada tanggal efektif, tanpa ubah kode |
| Payroll approved | Tidak berubah walau setting diubah (dibuktikan test snapshot) |
| Audit | 100% perubahan setting punya actor, alasan, nilai lama & baru |
| Default = sheet | Default seed menghasilkan angka = sheet pada fixture Sept/Okt 2026 |
| Penjelasan | Setiap angka di slip bisa ditelusuri ke setting & sumber scope-nya |

---

## 18. Aturan Persetujuan Dinamis (v2.2)

Syarat persetujuan **bukan lagi bawaan statis**. Owner dapat mengaturnya per pengaturan (atau per kategori):
- on/off
- role penyetuju
- jumlah penyetuju
- mode: *siapa pun dari role* / *minimal satu dari tiap role*
- masa kedaluwarsa
- berlaku per scope (kelompok gaji, lokasi, jabatan, karyawan) dan per tanggal

Setiap perubahan aturan persetujuan itu sendiri **wajib disetujui dan tercatat**.

### 18.1 Layar
- **Pengaturan Aturan → tab "Persetujuan"** pada setiap kartu setting:
  - status saat ini (On, Owner, 1 penyetuju, sumber "Kategori Bonus")
  - riwayat
  - tombol "Ubah aturan persetujuan" (khusus Owner)
- **Pengaturan → Kebijakan Persetujuan** (`/settings/approval-policies`): tabel semua key/kategori dengan kolom Persetujuan on/off, Role penyetuju, Min penyetuju, Mode, Scope, Berlaku dari–sampai, dan Floor (ikon gembok).
  - Filter "Persetujuan dimatikan" dan "Diubah 30 hari terakhir".
  - Form ubah: nilai baru, tanggal berlaku (≥ hari ini), alasan wajib. Peringatan muncul jika key ber-*floor* (perlu 2 penyetuju).
- **Kotak Persetujuan** (`/approvals`): daftar permintaan pending dengan detail sebelum/sesudah, simulasi dampak (§17.1), tombol Setujui/Tolak dengan catatan, dan progres "1 dari 2 penyetuju".
- **Audit log** (`/settings/audit`): filter tipe *Nilai setting / Kebijakan persetujuan / Keputusan*.

### 18.2 Default awal
| Pengaturan / kategori | Persetujuan | Penyetuju | Min | Floor |
|---|---|---|---|---|
| `feature.bpjs`, `feature.pph21`, parameter BPJS/PPh21 | On | Owner | 1 | ✅ |
| Kategori Bonus (tier, metode alokasi, bobot jabatan, bonus catering) | On | Owner | 1 | ✅ |
| `feature.salary_holdback` & aturan gantungan | On | Owner | 1 | ✅ |
| Kategori Tarif (gaji/pecahan gaji, tarif jam, lembur flat, borongan, fee kunjungan) | On | Owner | 1 | ✅ |
| Peran & akses (`rbac.*`) | On | Owner | 1 | ✅ |
| **Aturan persetujuan itu sendiri** (`meta.approval_policy`) | On (terkunci) | Owner + HR Admin | 2 (tiap role) | 🔒 |
| Presensi, telat, potongan izin/sakit, periode, mingguan pusat, shift, slip, sumber omset | Off (cukup audit) | — | — | — |
| `system.public_api` | Terkunci **mati**, tidak bisa diubah | — | — | 🔒 |

### 18.3 Pengaman
- Mengubah kebijakan persetujuan selalu butuh Owner dan dua orang (Owner + HR Admin). Tidak ada self-approval.
- Key ber-floor hanya bisa dimatikan/dilonggarkan dengan 2 penyetuju termasuk Owner, dan semua Owner/HR Admin mendapat notifikasi.
- Tidak berlaku mundur; permintaan yang sudah berjalan memakai aturan saat diajukan.
- Audit log tidak bisa diedit atau dihapus.
- Sistem menolak kebijakan yang membuat permintaan tidak bisa disetujui (role tanpa orang yang cukup).

### 18.4 Acceptance criteria
| Kriteria | Target |
|---|---|
| Owner mengubah approval satu setting | Berlaku sejak tanggal efektif, tanpa deploy |
| Mematikan approval BPJS | Butuh 2 penyetuju; muncul di audit & notifikasi |
| Self-approval | Selalu ditolak |
| `system.public_api` | Tidak muncul sebagai kontrol yang bisa diubah; API menolak 403 |
