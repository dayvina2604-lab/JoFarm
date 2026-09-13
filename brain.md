# 🧠 JOFARM FINANCE MINI APP - SYSTEM BLUEPRINT (BRAIN.MD)

> **Status:** Active Development (V2.0 - Multi-Wallet & Anti-Double-Input)  
> **Terakhir Diperbarui:** 13 September 2026  
> **Tujuan Dokumen:** Menjadi sumber kebenaran tunggal (*Single Source of Truth*) perancangan, aturan bisnis, skema basis data, dan dokumentasi API Jofarm Finance. Setiap perubahan pada basis data atau kode wajib dicatat pada dokumen ini.

---

## 1. Arsitektur Sistem
[ Telegram Client (Android / iOS / Desktop) ]
│
▼ (1. Render UI WebApp via SDK)
[ Frontend: GitHub Pages (Single Page Application) ]
• Framework : HTML5, Tailwind CSS, Vanilla JS
• Client AI : HTML5 Canvas Image Compression (~150KB)
│
▼ (2. HTTP POST text/plain to bypass CORS)
[ Backend API: Google Apps Script Web App (/exec) ]
• Engine   : V8 Runtime
• Auth     : RBAC via Telegram User ID
• AI OCR   : Gemini 2.5 Flash API (Multimodal Vision)
• Anti-Dup : 50-Row Sliding Window Matcher
├──► [ Google Drive Folder ] (Penyimpanan File Foto Nota)
└──► [ Google Sheets Master ] (Database Utama & Agregasi Saldo)

---

## 2. Struktur Hak Akses (RBAC) & Tim

Hak akses dikendalikan melalui tab `Users` di Google Sheets berdasarkan **ID Akun Telegram**:

| Nama Admin | Peran (Role) | Hak Akses Input | Hak Akses Laporan & Riwayat |
|---|---|---|---|
| **Superadmin - Owner** | `Superadmin` | Pemasukan & Pengeluaran | Seluruh data transaksi + Rekap Global seluruh admin + Filter per admin |
| **Oma Erna** | `Admin 1` | Pemasukan & Pengeluaran | Terbatas: Hanya transaksi miliknya sendiri |
| **Nando** | `Admin 2` | Pemasukan & Pengeluaran | Terbatas: Hanya transaksi miliknya sendiri |
| **Acha - AA** | `Admin 3` | Pemasukan & Pengeluaran | Terbatas: Hanya transaksi miliknya sendiri |
| **Kak Nana - Koko** | `Admin 4` | Pemasukan & Pengeluaran | Terbatas: Hanya transaksi miliknya sendiri |

---

## 3. Skema Basis Data (Google Sheets)

### A. Tab `Dashboard` (Eksekutif)
Menghitung saldo kas riil secara otomatis menggunakan formula native Google Sheets:
* **Cash in Hand (Tunai / Brankas / Laci):**
  `=SUMIFS(Transaksi!E:E, Transaksi!B:B, "Pemasukan", Transaksi!C:C, "Cash in Hand") - SUMIFS(Transaksi!E:E, Transaksi!B:B, "Pengeluaran", Transaksi!C:C, "Cash in Hand")`
* **Cash in Bank (Rekening / Transfer):**
  `=SUMIFS(Transaksi!E:E, Transaksi!B:B, "Pemasukan", Transaksi!C:C, "Cash in Bank") - SUMIFS(Transaksi!E:E, Transaksi!B:B, "Pengeluaran", Transaksi!C:C, "Cash in Bank")`
* **Total Kas Aktif:** `=Saldo Tunai + Saldo Bank`

### B. Tab `Transaksi` (Database Transaksi)
* Kolom A (1): `Tanggal` (`DD/MM/YYYY`)
* Kolom B (2): `Jenis` (`Pemasukan` | `Pengeluaran`)
* Kolom C (3): `Akun / Dompet` (`Cash in Hand` | `Cash in Bank`)
* Kolom D (4): `Kategori`
* Kolom E (5): `Nominal` (Numerik murni, format `Rp #,##0`)
* Kolom F (6): `Keterangan` (String)
* Kolom G (7): `Bukti Nota` (Formula `=HYPERLINK(url, "Lihat Bukti")`)
* Kolom H (8): `Input Oleh` (Nama Admin sesuai tab `Users`)
* Kolom I (9): `Waktu Input` (`DD/MM/YYYY HH:mm:ss`)

### C. Tab `Users` (Otorisasi)
* Kolom A: `User ID (Telegram)` (Angka unik)
* Kolom B: `Nama Admin`
* Kolom C: `Peran (Role)` (`Superadmin` | `Admin 1-4`)
* Kolom D: `Status` (`Aktif` | `Nonaktif`)

---

## 4. Aturan Bisnis & Logika Khusus

### 1. Sistem Multi-Dompet (Dual Wallet)
Setiap transaksi wajib menetapkan dompet:
* **Cash in Hand (Tunai):** Menambah/mengurangi kas fisik di kandang. Digunakan saat mencocokkan uang di laci kandang (*cash opname*).
* **Cash in Bank (Rekening):** Menambah/mengurangi kas bank. Digunakan saat mutasi rekening m-Banking.

### 2. Mekanisme Proteksi Anti-Double Input
Pencegahan pencatatan ganda berjalan dalam dua tahap:
1. **Frontend Button Lock:** Tombol *submit* langsung dinonaktifkan seketika saat diklik untuk mencegah duplikasi *double-tap* akibat latensi sinyal.
2. **Backend Duplicate Inspection (`checkDuplicate`):** Sebelum menyimpan, backend memindai 50 transaksi terakhir. Jika ditemukan transaksi dengan **Tanggal, Jenis, Akun, Kategori, dan Nominal yang identik**, backend mengembalikan `isDuplicate: true` beserta detail nama penginput terdahulu. Frontend wajib menampilkan dialog konfirmasi kepada pengguna sebelum memaksakan penyimpanan (`forceSave`).

### 3. Pemindaian Nota via Gemini 2.5 Flash
* Gambar dikompresi di sisi browser melalui Canvas menjadi format JPEG (maksimal lebar 1000 px, kualitas 0,7) sebelum dikirim ke backend.
* Backend mengirim Base64 ke Gemini 2.5 Flash API dengan *System Instruction* berformat JSON terstruktur.
* Nilai tanggal, jenis, akun, kategori, nominal, dan keterangan otomatis mengisi formulir (*auto-fill*).

---

## 5. Spesifikasi Kontrak API (`doPost`)

Setiap *request* wajib berformat JSON dan dikirimkan dengan header `Content-Type: text/plain;charset=utf-8` untuk menghindari blokir CORS Preflight Apps Script.

### 1. `getInitialData`
* **Payload:** `{ "userId": "6853244356" }`
* **Response:**
  ```json
  {
    "success": true,
    "user": { "id": 6853244356, "name": "Superadmin - Owner", "role": "Superadmin" },
    "balances": { "cashInHand": 1250000, "cashInBank": 14500000, "totalKas": 15750000 },
    "adminList": ["Superadmin - Owner", "Oma Erna", "Nando", "Acha - AA", "Kak Nana - Koko"]
  }
2. checkDuplicate
Payload: { "tanggal": "2026-09-13", "jenis": "Pemasukan", "akun": "Cash in Bank", "kategori": "Top Up Saldo", "nominal": 1000000 }

Response (Jika Duplikat):

JSON
{
  "success": true,
  "isDuplicate": true,
  "existing": {
    "tanggal": "13/09/2026",
    "nominal": 1000000,
    "kategori": "Top Up Saldo",
    "inputBy": "Oma Erna",
    "timestamp": "13/09/2026 09:12:00"
  }
}
3. saveTransaction
Payload:

JSON
{
  "tanggal": "2026-09-13",
  "jenis": "Pengeluaran",
  "akun": "Cash in Hand",
  "kategori": "Pakan",
  "nominal": 350000,
  "keterangan": "Beli dedak 2 sak",
  "adminName": "Nando",
  "imageBase64": "data:image/jpeg;base64,...",
  "imageMime": "image/jpeg"
}
4. getReport
Payload:

JSON
{
  "userId": "6853244356",
  "startDate": "2026-09-01",
  "endDate": "2026-09-13",
  "selectedAdmin": "Semua"
}
6. Riwayat Perubahan (Changelog)
[V2.0] - 13 September 2026
Added: Pemisahan dompet multi-akun (Cash in Hand vs Cash in Bank).

Added: Kartu ringkasan saldo fisik vs saldo bank di tab Dashboard.

Added: Endpoint backend checkDuplicate untuk mitigasi double input pemasukan/pengeluaran antar-admin.

Added: Pendaftaran admin ke-4: Admin 4 - Kak Nana - Koko.

Added: File pedoman arsitektur sistem (brain.md).

[V1.0] - Awal September 2026
Inisialisasi Telegram Mini App dengan basis GitHub Pages & Google Apps Script.

Integrasi awal OCR struk nota otomatis menggunakan Gemini API.

Desain tabel warna Forest Emerald.


---

### Langkah Berikutnya
1. Tempel isi `Setup.gs` yang baru ke editor Apps Script, lalu jalankan fungsi `setupJofarmSpreadsheet`.
2. Ganti file `Code.gs` Anda dengan kode backend baru di atas, lalu deploy versi baru (**Deploy > Manage deployments > Edit > New version > Deploy**).
3. Buat file `brain.md` di repositori GitHub Anda dan tempel isinya sebagai acuan dokumentasi.
