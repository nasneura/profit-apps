# PROFIT — Project Memory

**PROFIT** = Project Financial Tracking. Web-app untuk monitor & keluarkan table/summary PnL & Cashflow per project, dengan consolidated view merentasi semua project, dan cashflow forecasting.

Fail utama: `pnl-cashflow-tracker.html` — single-file HTML app (vanilla JS, no framework), backend Supabase.

---

## 1. Backend (Supabase)

- **Project ref:** `zgewkcvgseonddybfckb`
- **Project name:** `pnl-cashflow-tracker`
- **Region:** ap-southeast-1 (Singapore)
- **Organization:** nas.neural
- **URL:** `https://zgewkcvgseonddybfckb.supabase.co`
- Anon/publishable key di-hardcode dalam `index.html` (`SUPABASE_URL`, `SUPABASE_KEY`) — ini normal untuk client-side app, security dikawal oleh RLS.

### Schema

**`profiles`** — extends `auth.users`
- `id`, `email`, `full_name`, `role` (`admin` | `member`, default `member`), `created_at`
- Auto-created via trigger `handle_new_user()` bila user sign up

**`projects`**
- `id`, `name`, `description`, `opening_balance` (numeric, default 0 — untuk cashflow forecast), `created_by`, `created_at`
- Hanya **admin** boleh create/edit/delete (RLS)

**`uploads`** — metadata setiap fail yang di-upload
- `id`, `project_id`, `statement_type` (`pnl`|`cashflow`), `file_name`, `period_label`, `column_mapping` (jsonb), `uploaded_by`, `uploaded_at`
- `file_name` boleh null (untuk manual entry yang tak lalui upload)

**`line_items`** — data sebenar PnL/Cashflow
- `id`, `upload_id` (nullable — null untuk manual entry), `project_id`, `statement_type`, `category`, `subcategory` (nullable), `amount`, `period` (date, nullable — bulan), `status` (`actual`|`forecast`, default `actual`), `created_by`, `notes`, `created_at`

### RLS model (permission)
- Semua authenticated user: boleh **view** semua projects & line items, boleh **upload/insert/update** line items
- Hanya **admin**: boleh create/edit/delete **projects**
- Delete line items: admin, ATAU creator (`created_by`), ATAU uploader asal
- Admin pertama: `nas.neura@gmail.com` — di-upgrade manual guna SQL (`update profiles set role='admin'`)

---

## 2. Struktur Data — Peraturan Penting

- **Category** = tahap atas/grouping (contoh: `Revenue`, `Cash-In`, `Expenses`, `Cash-Out`) — BUKAN nama item spesifik
- **Subcategory** = item spesifik (contoh: `Direct Manpower`, `Client Payment`, `Travel & Accomodation`)
- Ini kadang tersalah semasa import (item jadi Category, Type jadi Subcategory) — ada butang **"Fix Cashflow structure"** dalam tab Uploads (admin only) untuk swap data sedia ada
- PnL dan Cashflow **tiada breakdown wajib ikut bulan** secara struktur, tapi setiap line item **boleh** ada `period` (bulan) — optional
- Sorting category: **ikut sign amount sebenar** (positive/duit masuk dulu, negative/duit keluar lepas), BUKAN ikut keyword nama category. Function: `sortCategoriesLogically()`
- Row striping (alternate warna) dalam List view: **ikut bulan** (`period`), bukan ikut posisi baris. Toggle warna setiap kali `period` berubah berbanding row sebelumnya. Function di `StatementTable()` — guna `stripeMonthKey`/`stripeOn` local variables, inline `style="background:var(--stripe)"`. **JANGAN** tambah CSS `nth-child(even)` alternation sebab akan berlanggar dengan logic ni (pernah jadi bug sebelum ni).

---

## 3. Features yang Dah Dibina

### Auth
- Email/password sign up & login (Supabase Auth)
- Password show/hide toggle (eye icon)
- Role: `admin` (create/edit project, semua akses) vs `member` (upload/view/edit line items sahaja)

### Projects
- Sidebar: senarai project + "Consolidated" (default view lepas login)
- New/Edit project: nama, description, opening cash balance
- Edit project: pensel icon di header (admin only)

### Line Items — 3 cara masuk data
1. **Manual satu-satu**: "+ Add line item" — Category (quickpick dropdown + text input untuk baru), Subcategory (sama), Amount, Month, Status (Actual/Forecast)
2. **Quick Fill**: "+ Quick fill months" — set Category+Subcategory sekali, isi amount untuk 12 bulan (tahun semasa) sekaligus, ada "Apply to all 12" untuk amount sama rata
3. **Upload Excel/CSV**: dua format template disokong —
   - **Flat**: `Category | Subcategory | Amount` (fail `PnL_Cashflow_Template.xlsx`)
   - **Matrix/Monthly**: `Type | Category | Jan..Dec | Total` (fail `PnL_Cashflow_Template_Monthly.xlsx`) — auto-detect, auto sign-flip untuk outflow (Cash-Out/COGS/Expense)
   - Ada langkah "pilih baris header" untuk fail yang tak standard, dan "exclude keywords" untuk skip baris subtotal/balance

### Views (per project)
- **Summary** — breakdown by category+subcategory, net PnL & Cashflow
- **PnL / Cashflow tabs** — dua mode:
  - **List** (default sebelum ni; ada toggle) — flat table, boleh Edit/Delete setiap row
  - **Grid** (default sekarang) — matrix Category/Subcategory × Bulan × Total, read-only (edit kena guna List)
- **Forecast** (Cashflow) — Opening balance + Actual + Forecast → running balance projection bulanan
- **Uploads** — history fail yang di-upload, admin dapat "Data maintenance" section
- **Consolidated** — jumlah semua project sekali gus

### Lain-lain
- Filter bulan (dropdown "All months" / bulan spesifik) — disorok bila Grid view aktif
- Confirmation modal (custom, bukan native `confirm()`) untuk semua Delete DAN Edit/Save actions
- Draft auto-save — setiap form (Add/Edit line item, Quick Fill, New/Edit Project) auto-simpan ke localStorage + sync live ke state, restore automatik bila page reload/tab discard (mobile). Banner "We restored your unsaved changes..." bila restore berlaku.

---

## 4. Design System

- **Font:** Lexend (semua — heading, body, angka guna `font-variant-numeric: tabular-nums`, bukan monospace)
- **Palette** (ikut project "TRACE" reference, bukan design asal "Ledger"):
  - `--ink:#101A2C` (navy gelap, sidebar/text)
  - `--teal:#2F8F86` (positive/accent)
  - `--rust:#B23B3B` (negative)
  - `--gold:#B96E17` (highlight/admin badge)
  - `--paper:#F7F5EF`, `--paper-dim:#EFEBE0`, `--stripe:#ECECE9` (row alternate — neutral grey, bukan cream)
  - `--navy-light:#E3E9F1` (background Category row dalam table — sengaja beza dari stripe subcategory row untuk kontra jelas)
  - `--green:#4C8C5C` (belum banyak dipakai, reserved)
- Table `.ledger`: header navy dengan teks putih uppercase, Category row guna `--navy-light`, subcategory row alternate putih/`--stripe` **ikut bulan**
- Nama app: **PROFIT**, tagline "Project Financial Tracking"

---

## 5. Nota Sejarah / Isu yang Pernah Timbul (elak ulang)

- **Bug MatrixGridView**: kesilapan kira jumlah per-bulan untuk baris Category & Grand total (rujuk struktur data salah — `cell.sum` patut `monthMap[c].sum`). Dah fixed.
- **Data hilang semasa isi form**: punca (a) Supabase `TOKEN_REFRESHED` event trigger full re-render tanpa guard, (b) state tak sync live semasa taip (cuma di submit). Fixed dengan: skip render() untuk `TOKEN_REFRESHED`/`INITIAL_SESSION`, dan sync `state.xxxModalData` terus dalam setiap `input` event handler (bukan cuma di submit).
- **Auto-detect upload template palsu-positif**: safety-check "nampak macam tab arahan" pernah match SEMUA sheet (sebab frasa sama wujud di semua tab). Fixed dengan tighten keyword match.
- **Str_replace pernah accidentally delete function declaration** (contoh `function UploadsView(){` hilang) semasa edit bersebelahan — sentiasa verify dengan `node --check` lepas edit besar.
- **Row striping**: pernah tersilap tukar ke CSS `nth-child(even)` (posisi baris) bila user sebenarnya nak ikut bulan — dua sistem tu **tak boleh** wujud serentak, akan berlanggar.

---

## 6. Preferensi Kerja Person Ni

- Bahasa: **Melayu** (kadang bercampur English untuk istilah teknikal)
- Suka **soalan jelas** bila permintaan ambiguous — sama ada nak ubah **spesifik satu bahagian** atau **konsisten semua page**
- Suka **verify dengan tool sebenar** (bukan cuma anggapan) — contoh: guna Node.js `--check` untuk verify syntax, test data sebenar sebelum claim "dah fix"
- App perlu nampak **profesional/enterprise-grade** — banyak rujukan dari NetSuite & reference app "TRACE" (React/Tailwind, PFM/PM role-based dashboard, warna navy/teal/amber)
- Fail selalu di-download & test dalam browser sendiri (bukan artifact preview) — jadi setiap perubahan kena `present_files` fail HTML penuh

---

## 7. Fail Berkaitan

- `pnl-cashflow-tracker.html` — app utama
- `PnL_Cashflow_Template.xlsx` — template flat (Category/Subcategory/Amount)
- `PnL_Cashflow_Template_Monthly.xlsx` — template matrix (Type/Category/bulan/Total)

---

*Kemaskini terakhir: rujuk timestamp conversation. Sila update file ni bila ada perubahan besar (schema, feature baru, design system) supaya session akan datang boleh terus faham konteks tanpa ulang tanya.*
# Staging environment
