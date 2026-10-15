# Architecture — Enclavitzy

## 1. Overview

**Enclavitzy** menggunakan arsitektur aplikasi desktop 2-tier modern yang mengombinasikan keandalan sistem native berkinerja tinggi dengan antarmuka web yang fleksibel:
- **Frontend** (React 18 + CodeMirror 6 + TypeScript + Tailwind CSS) — UI interaktif, editor teks virtualized, state management client, dan visualisasi status kompetisi.
- **Backend** (Rust 1.77+ + Tauri v2) — Manajemen proses kompilasi native C++, HTTP client & session scraping DOMjudge, I/O sistem berkas lokal, keamanan kiosk level OS, dan process sandbox.

Tujuan utama migrasi ke arsitektur ini adalah **menghilangkan beban Chromium bundle Electron** sehingga memori idle turun drastis ke **~30–40 MB** (dari 200–300 MB) dan startup time menjadi **< 1 detik**.

---

## 2. Frontend Layer

### 2.1 Tech Stack
- **UI Framework:** React 18
- **Bahasa:** TypeScript strict (`noImplicitAny: true`, `noUncheckedIndexedAccess: true`)
- **Code Editor:** **CodeMirror 6** via `@uiw/react-codemirror` dan `@codemirror/lang-cpp` (~140 KB gzipped, runtime memory ~12 MB)
- **Styling:** Tailwind CSS 3.4 + komponen bergaya shadcn/ui (dark contest palette)
- **Animasi:** GSAP 3 (`@gsap/react`, `useGSAP()`) — dibatasi hanya properti `transform` dan `opacity` untuk performa stabil 60fps
- **Icons:** `lucide-react`

### 2.2 State Management
```
┌────────────────────────────────────────────────────────┐
│  Zustand 5 (Client State)                              │
│  - auth.store: username, teamId, loginStatus           │
│  - editor.store: currentCode per problem, unsavedDirty │
│  - ui.store: activeProblemId, activeTab, proctorModal  │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│  TanStack Query 5 (Server State & Polling)             │
│  - useContests (cacheTime 60s)                         │
│  - useProblems (cacheTime 60s)                         │
│  - useSubmissions (pollInterval 3000ms)                │
│  - useVerdicts (pollInterval 3000ms, stop saat final)  │
│  - useClarifications (pollInterval 3000ms)             │
│  - useScoreboard (pollInterval 10000ms, extract state) │
└────────────────────────────────────────────────────────┘
```

### 2.3 Dynamic Contest Discovery (Zero-Redundancy Configuration)
- **Eliminasi Hardcoded Contest ID:** Berkas konfigurasi lokal (`contest.config.json` dan `.env`) tidak memuat `contest_id` untuk mencegah redundansi dan kekakuan multi-kontes.
- **Discovery via API:** Saat peserta memasukkan user & pass, frontend memanggil `GET /api/v4/contests` dengan Basic Auth untuk memperoleh kontes terotorisasi dari DOMjudge.
- **Routing Otomatis:**
  - **1 Kontes:** Langsung *auto-enter* ke kontes tersebut.
  - **> 1 Kontes:** Menampilkan layar seleksi kontes interaktif.
- **Session RAM:** ID kontes yang dipilih disimpan secara eksklusif di `session.contest_id` di memori RAM runtime.

### 2.4 Clean Contest Switch & C++ Workspace Isolation Lifecycle
- **Auto-Flush Sebelum Berpindah:** Jika ada penundaan debounce auto-save (`saveTimeoutRef`), sistem segera melakukan commit/flush berkas aktif ke kontes saat ini agar perubahan kode tidak hilang.
- **Store & Cache Invalidation:**
  - `useEditorStore.resetContestState()`: Mengosongkan `codeMap`, `samples`, `sampleResults`, `compileResult`, dan `customStdin`.
  - `clearStatementMemoryCache()`: Menghapus cache memory PDF soal agar tidak terjadi tumpang tindih soal antar kontes.
  - Reset state UI: `verdictMap`, `trackedSubmission`, `submissionFeedback`, dan penanda klarifikasi dibersihkan.
- **Backend Directory & Volatile Sync:**
  - `domjudge_set_active_contest` menyelaraskan `s.contest_id` dan `s.contest_name` (nama kontes kanonikal `name` || `formal_name` || `shortname`).
  - Mengosongkan `statement_ram_cache` dan `local_submissions` di backend Rust `AppState`.
  - Direktori kerja lokal (`Desktop/Enclavitzy/Submissions/{hash}_{contest_name}/`) di-resolve secara independen, sehingga kontes baru memuat berkas C++ asli miliknya dari disk atau template baru.

### 2.5 C++ Auto-Formatter & Beautifier Engine (`src/lib/cppFormatter.ts`)
- **Zero-Dependency Pure Tokenizer:** Dibangun dengan TypeScript murni (< 5 KB) tanpa memerlukan runtime compiler eksternal (LLVM/WASM/ClangFormat) agar memenuhi batasan installer < 10 MB dan RAM < 100 MB.
- **Hierarchical Indentation (4 Spasi):** Mengalkulasi kedalaman blok `{ }`, `if`, `for`, `while`, dan `switch` secara presisi.
- **Inline Initializer List Preservation:** Mengenali kurung kurawal tanpa titik koma `{u, v}` atau `{1, 2, 3}` untuk dipertahankan inline.
- **Multi-Statement Line Splitting:** Memecah beberapa statement dalam satu baris (`int n; cin >> n;`) menjadi baris terpisah, sembari menjaga semicolon pada header `for (int i = 0; i < n; ++i)`.
- **Literal & Preprocessor Safety:** Mempertahankan string literal (`"..."`), karakter (`'...'`), komentar inline/blok, dan direktif `#include`/`#define` tetap di kolom 0 tanpa modifikasi destruktif.
- **Shortcut & Telemetry:** Terintegrasi dengan tombol Magic Wand (`Wand2`), shortcut `Shift+Alt+F` (standar IDE) dan `Ctrl+Shift+F` (aman Kiosk), serta diklasifikasikan sebagai `NORMAL` telemetry event.

---

## 3. Backend Layer (Rust / Tauri v2)

### 3.1 Struktur Modul (`src-tauri/src/`)
```
src-tauri/src/
├── main.rs                 # Inisialisasi Tauri builder & lifecycle
├── lib.rs                  # Registrasi plugin, command handlers, prevent_close & kiosk setup
├── commands/               # IPC handlers (#[tauri::command])
│   ├── mod.rs
│   ├── config.rs           # config_get, config_get_template
│   ├── compiler.rs         # compile_and_run, compile_only
│   ├── workspace.rs        # load_problem_file, save_problem_file, list_files
│   ├── samples.rs          # fetch_samples, run_all_samples
│   ├── materials.rs        # materials_get_list, materials_get_pdf (anti-tamper SHA-256)
│   ├── domjudge.rs         # login, submit, poll_verdict, clarifications, get_statement, scoreboard
│   └── proctor.rs          # verify_pin, exit_kiosk (uninstall hook + exit)
├── services/               # Business logic & external systems
│   ├── config_loader.rs    # Parsing & validasi contest.config.json
│   ├── materials_service.rs# Whitelist manifest & verifikasi integritas SHA-256 materi
│   ├── compiler_resolver.rs# Deteksi g++.exe (bundled → common → PATH)
│   ├── compiler_runner.rs  # tokio::process::Command runner + timeout + cleanup
│   ├── workspace_manager.rs# Isolasi Desktop/Enclavitzy/Submissions & scratchpad files
│   ├── samples_service.rs  # Scraper zip + runner CPH lokal
│   ├── domjudge_client.rs  # REST API v4 client + Basic Auth + statement fetcher
│   └── session_manager.rs  # Cookie jar persistent (reqwest_cookie_store)
├── security/               # Layer keamanan & isolasi
│   ├── kiosk.rs            # Low-level keyboard hook (WH_KEYBOARD_LL) & SetWindowDisplayAffinity
│   ├── sandbox.rs          # Job Objects (Win) / rlimit (Unix)
│   └── headers.rs          # Stripping CSP / frame restrictions
├── state/
│   └── app_state.rs        # Arc<RwLock<...>> global state & statement_ram_cache
├── error.rs                # Definisi error terstruktur (thiserror)
└── utils/
    ├── logger.rs           # Structured logging (tracing-subscriber)
    └── paths.rs            # Resolusi path dev vs packaged
```

### 3.2 Model State Backend (`AppState`)
```rust
pub struct AppState {
    pub config: Arc<RwLock<ContestConfigFile>>,
    pub session: Arc<RwLock<ActiveSession>>,
    pub domjudge: Arc<DomjudgeClient>,
    pub statement_ram_cache: Arc<RwLock<HashMap<String, String>>>, // Volatile RAM-only PDF cache
    pub local_submissions: Arc<RwLock<Vec<Value>>>,               // Session in-memory submission tracking
}
```

---

## 4. Alur Kompilasi, Eksekusi C++ & Pre-Compilation Guard

Sistem mendukung dua kategori berkas:
1. **Solusi Soal Kompetisi (`{problemId}.cpp`):** Terhubung ke alur submit DOMjudge, local test runner samples, dan verdict polling.
2. **Scratchpad TMP (`scratch_1.cpp`, `scratch_2.cpp`, `scratch_3.cpp`):** Lembar kerja mandiri untuk coret-coretan/hitung pola/test kode. Bisa di-compile (`F9`) dan run (`F11`), tetapi tombol Submit dinonaktifkan (`canSubmit = false`).

### 4.1 Pre-Compilation Guard Sebelum Submit
```
Peserta klik "Kirim Solusi"
          ↓
1. Autosave kode ke disk
2. Jalankan api.compileOnly(problemId, code) via native MinGW GCC
          ↓
[Kompilasi Gagal?] ──Ya──> Tampilkan modal konfirmasi (CompileWarningModal):
          │                - Tampilkan diagnostik error compiler
          │                - Tombol "Batal Kirim" / "X": Tutup modal, batal kirim
          │                - Tombol "Tetap Lanjutkan Kirim": Eksekusi submit
        Tidak
          ↓
### 4.2 Local Execution Timeout (21 Detik) & Force Stop (STOP Button)
- **Time Limit Lokal:** Ditetapkan default **21 detik** (`run_timeout_secs: 21` di `contest.config.json`). Ini memberi ruang leluasa bagi program dengan kalkulasi intensif atau I/O besar tanpa terkena TLE terlalu dini.
- **Force Stop Process Management:**
  - Saat binary dijalankan, Backend Rust mencatat Process ID (`child.id()`) ke `state.active_run_pid`.
  - Frontend mengubah tombol "Jalankan (F11)" menjadi tombol merah mencolok **"STOP"**.
  - Menekan tombol "STOP" (atau shortcut F11 saat running) memanggil command `stop_execution`.
  - Backend Rust mengeksekusi terminasi paksa process tree (`taskkill /F /T /PID {pid}` di Windows / `kill -9` di Unix) dan membersihkan sisa binary seketika.
  - Peserta langsung menerima status diagnostik: `Program dihentikan paksa oleh pengguna (STOP / Force Terminated)`.

---

## 5. Alur Ephemeral RAM Statement Cache, Blob URL & Hard Reload

Untuk mencegah kebocoran soal pada PC lab, berkas soal PDF tidak pernah disimpan ke disk:

```
Peserta buka tab problem atau klik "Hard Reload"
          ↓
invoke('domjudge_get_statement', { problemId, forceRefresh })
          ↓
Backend Rust (domjudge.rs):
1. Jika forceRefresh == false:
   Cek state.statement_ram_cache.read()
   → Jika ada, langsung return Base64 Data URL (0ms latency, zero I/O)
2. Jika belum ada atau forceRefresh == true:
   a. GET /api/v4/contests/{cid}/problems/{problemId}/statement (Basic Auth, timeout 60s)
   b. Terima response bytes langsung di memori RAM
   c. Encode bytes ke Base64 Data URL: "data:application/pdf;base64,..."
   d. Simpan ke state.statement_ram_cache.write()
   e. Return Data URL ke frontend
          ↓
Frontend:
1. Ubah Base64 string menjadi binary Uint8Array -> Blob di memori RAM
2. Buat URL.createObjectURL(blob) (menghindari batas panjang URL iframe WebView2 pada file PDF berukuran megabytes)
3. Render via <object data="blob:...#toolbar=0" type="application/pdf"> dengan fallback <iframe>
4. Bersihkan memori dengan URL.revokeObjectURL saat ganti soal / unmount
```
> **Garansi Keamanan:** Begitu aplikasi ditutup, force killed, atau komputer lab dimatikan, seluruh memori RAM terbebas dan tidak ada berkas PDF yang tertinggal di hard disk.

---

## 5.1 Alur Materi & Modul Referensi (Anti-Tamper & Embedded Bundling)

Fitur ini menyediakan modul referensi materi kompetisi/praktikum (misal: Modul Struktur Data & Algoritma, Modul STRAGO) yang dapat diakses peserta langsung dari tab navigasi mandiri `Materi` di ZenSidebar:

```
invoke('materials_get_list')
          ↓
Backend Rust (materials_service.rs):
1. Mengembalikan daftar materi resmi (ID, judul, deskripsi, sha256 checksum, ukuran byte) dari static manifest.
2. Mencegah manipulasi daftar oleh peserta (tidak memindai sembarang file di disk).

invoke('materials_get_pdf', { id: 'sda' })
          ↓
Backend Rust:
1. Validasi ID terhadap approved whitelist ('sda', 'strago').
2. Ambil byte PDF dari binary payload (include_bytes!) atau bundle resource terverifikasi.
3. Validasi kriptografi SHA-256 terhadap hardcoded checksum resmi:
   - SDA: 810e4c4866c054a1566626a30103737f84bf63dbf3931db2bb4c496e86c84bed
   - STRAGO: 3f81a702a794fcb236bbad74457bd976c8c6b67e5e727d50753b63509c35b2a5
4. Jika checksum cocok, encode ke base64 data URL ("data:application/pdf;base64,...").
5. Jika checksum tidak cocok atau file dimanipulasi, return AppError::Security ("Integrity check failed").
          ↓
Frontend:
1. Memuat PDF via `pdfjs-dist` langsung ke elemen HTML5 `<canvas>`.
2. **In-Document Search Engine (Ctrl+F):** Mengekstrak `textContent` per halaman via `page.getTextContent()`, melakukan pencocokan teks case-insensitive, menghitung jumlah kecocokan global, menyediakan navigasi next/prev match, auto-scroll ke halaman yang cocok, dan highlight kontainer halaman. Terintegrasi dengan pencegah browser Find bawaan.
3. Anti-leak print ban (`@media print { display: none }`), tanpa tombol download/ekspor, proteksi context menu.
```

## 6. Alur Local Test Runner (Samples Scraper & CPH)

```
invoke('fetch_samples', { probid })
          ↓
Backend Rust (samples_service.rs):
1. Cek cache lokal di memori
2. Jika belum ada:
   a. GET https://<domjudge>/login → parse _csrf_token
   b. POST https://<domjudge>/login (_username, _password, _csrf_token)
   c. GET https://<domjudge>/team/{probid}/samples.zip (menggunakan cookie jar)
   d. Validasi Content-Type = application/zip (bukan HTML 404)
   e. Unpack zip via crate `zip`
   f. Pasangkan berkas *.in dan *.ans menjadi Vec<Sample>
3. Return Vec<Sample> ke frontend
          ↓
Frontend: Tombol "Run All Samples" → invoke('run_all_samples', { problemId, sourceCode, samples })
          ↓
Backend mengeksekusi binary terhadap setiap input sample dan membandingkan output:
- Normalisasi baris (\r\n vs \n, trailing whitespace)
- Return Vec<SampleRunResult> dengan status PASS / FAIL
```

---

## 7. Alur Submit & Polling Verdict DOMjudge

```
invoke('submit_solution', { problemId, languageId, sourceCode })
          ↓
[Validasi Frontend & Backend]: Pastikan problemId BUKAN "scratch_*"
          ↓
Backend Rust (domjudge_client.rs):
1. POST /api/v4/contests/{cid}/submissions (multipart form)
2. Header: Authorization Basic base64(user:pass)
3. Return submit_id (misal: "142")
          ↓
Frontend memulai interval polling (TanStack Query):
- Setiap 1s (15x pertama) lalu 3s (10x berikutnya):
  invoke('poll_verdict', { submissionId: "142" })
          ↓
Backend: GET /api/v4/contests/{cid}/judgements?submission_id=142
          ↓
Response Judgement:
- Jika verdict == "CE" & max_run_time == null:
  Tampilkan badge Compile Error + tombol bantuan buka Web UI (/team/submission/142)
- Jika verdict final (AC, WA, TLE, MLE, RTE):
  Hentikan polling dan tampilkan badge verdict
```

---

## 8. Arsitektur Keamanan Berlapis (Defense in Depth)

```
┌────────────────────────────────────────────────────────────────────────┐
│ Layer 1: Tauri Capabilities (src-tauri/capabilities/default.json)      │
│ - IPC commands whitelist (default deny)                                │
│ - Scope fs: terbatas pada $DESKTOP/Enclavitzy/*                        │
│ - Scope shell: terbatas pada binary g++.exe                            │
│ - CSP: default-src 'self'; connect-src https://host; frame-src data:; │
└────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌────────────────────────────────────────────────────────────────────────┐
│ Layer 2: Rust Memory & Zero-Disk Ephemeral Cache                       │
│ - Tidak ada buffer overflow, null pointer, race conditions             │
│ - Statement PDF disimpan eksklusif di RAM (statement_ram_cache)        │
│ - Kredensial Basic Auth disimpan di RAM, musnah saat shutdown          │
└────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌────────────────────────────────────────────────────────────────────────┐
│ Layer 3: Runtime Kiosk Lockdown, UAC Manifest & Hardware Display Guard │
│ - UAC App Manifest: otomatis elevasi requireAdministrator saat dibuka  │
│ - SAS (Ctrl+Alt+Del) Lockdown: NoLogoff, HideFastUserSwitching, NoClose│
│ - Window fullscreen mutlak, alwaysOnTop, decorations: false            │
│ - Win32 Low-Level Hook (WH_KEYBOARD_LL): block Alt+Tab, Alt+F4, WinKey │
│ - SetWindowDisplayAffinity: proteksi anti-screenshot & OBS screen share│
│ - Strict Whitelist External App Guard: deteksi & kill semua app luar   │
│ - WindowEvent::CloseRequested: pencegatan prevent_close mutlak         │
│ - Exit window hanya via verifikasi Proctor PIN sah di backend Rust     │
└────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌────────────────────────────────────────────────────────────────────────┐
│ Layer 4: Process Sandbox untuk Binary Peserta                          │
│ - Windows: Windows Job Object (CPU time & memory limits)               │
│ - Linux/macOS: rlimit (RLIMIT_AS, RLIMIT_CPU, RLIMIT_FSIZE)            │
│ - Network disabled untuk child process binary peserta                  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Distribusi & Packaging (Dual Build)

Konfigurasi `src-tauri/tauri.conf.json` mendukung pembuatan dua varian distribusi Windows sekaligus:
- **`.msi` Installer (WiX Toolset):** Ditujukan untuk administrator lab komputer sekolah/kampus guna instalasi serentak via Active Directory / Group Policy Object (GPO).
- **`.exe` Portable / Standalone (NSIS):** Ditujukan untuk peserta atau instalasi cepat tanpa hak administrator (zero-install portable runner).

Konfigurasi build:
```json
{
  "bundle": {
    "active": true,
    "targets": ["msi", "nsis"]
  }
}
```

---

## 10. Target & Karakteristik Kinerja

| Metrik | Baseline (v2 Electron) | Target (v3 Tauri) | Pengukuran Aktual |
|--------|------------------------|-------------------|-------------------|
| Ukuran Installer | ~120 MB | < 10 MB | **~8–10 MB** |
| Penggunaan RAM (Idle) | ~220 MB | < 50 MB | **~30–40 MB** |
| Penggunaan RAM (Editor + Webview) | ~350 MB | < 120 MB | **~70–95 MB** |
| Waktu Cold Startup | ~2.4 detik | < 1.0 detik | **~0.6–0.8 detik**|
| Inisialisasi Editor | ~1200 ms | < 500 ms | **~350–400 ms** |
| Kecepatan Kompilasi (Single File) | ~2.5 detik | < 2.0 detik | **~1.2–1.6 detik**|
| Disk Leakage (Statement PDF) | File tersimpan di disk | 0 Bytes di disk | **0 Bytes (RAM only)** |
