# Security & Lockdown Model — Save Exam UPC

## 1. Overview
Save Exam UPC dirancang untuk kompetisi pemrograman on-site dengan pengawasan ketat. Keamanan berfokus pada:
1. **Desktop Lockdown (Kiosk Mode)** — Mencegah peserta berpindah aplikasi, membuka browser eksternal, mengambil tangkapan layar, atau mengakses file sistem di luar direktori lomba.
2. **Low-Level Hardware & OS Guard** — Memblokir shortcut tingkat rendah (`Alt+Tab`, `Alt+F4`, `WinKey`, `PrintScreen`) via Win32 Low-Level Keyboard Hook (`WH_KEYBOARD_LL`) serta mematikan capture layar via `SetWindowDisplayAffinity`.
3. **Pencegahan Force Close & Window Protection** — Mencegat event penutupan window (`prevent_close`), di mana aplikasi hanya dapat ditutup via verifikasi PIN Pengawas di backend Rust.
4. **Zero-Disk Leakage (Ephemeral RAM Statement Cache)** — Berkas soal PDF tidak pernah ditulis ke disk; disimpan eksklusif di memori RAM dan musnah 100% saat aplikasi ditutup atau komputer mati.
5. **IPC & Capability Isolation (Tauri v2)** — Menolak semua hak akses yang tidak dibutuhkan secara eksplisit (*least privilege*).
6. **Execution Sandbox** — Membatasi konsumsi memori, waktu CPU, dan melarang akses jaringan pada binary yang dikompilasi dari kode peserta.
7. **Scratchpad Isolation** — Berkas coret-coretan mandiri (`scratch_1..3.cpp`) terisolasi dari antrean submission DOMjudge.

---

## 2. Kiosk Mode & 4-Layer Desktop Lockdown

### 2.1 Konfigurasi Window (`tauri.conf.json`)
```json
{
  "app": {
    "windows": [
      {
        "title": "Save Exam UPC",
        "fullscreen": true,
        "alwaysOnTop": true,
        "decorations": false,
        "resizable": false,
        "skipTaskbar": false
      }
    ]
  }
}
```

### 2.2 Layer 1: Time-Critical Low-Level Keyboard Hook (`WH_KEYBOARD_LL`) & Safe Thread Watchdog
Di tingkat OS Windows (aktif saat Kiosk Mode), backend Rust memasang hook keyboard global berkinerja tinggi:
- **Pra-inisialisasi Message Queue:** Memanggil `PeekMessageW(..., PM_NOREMOVE)` sebelum `SetWindowsHookExW` untuk memastikan thread memiliki message queue yang valid.
- **Prioritas Time-Critical & Zero Overhead (<20ns):** Thread hook diangkat ke `THREAD_PRIORITY_TIME_CRITICAL` (15). Callback dieksekusi secara instan (< 20ns) tanpa syscall, tanpa log, tanpa mutex, dan tanpa IPC channel di dalam callback guna mencegah pemicuan silent unhook Windows (`LowLevelHooksTimeout`).
- **Pelacakan Modifier Murni In-Memory & Asinkron Hardware:** Status tombol `Win`, `Alt`, `Ctrl`, dan `Shift` dilacak via atomics (`AtomicBool`) dikombinasikan dengan `GetAsyncKeyState` yang membaca status perangkat keras fisik secara langsung tanpa terikat message queue thread.
- **Penelanan Total Tombol Windows, Alt, Escape & Context Menu:**
  - Tombol `VK_LWIN` (`0x5B`), `VK_RWIN` (`0x5C`), `VK_APPS` (`0x5D`), `VK_MENU` (`0x12`), `VK_LMENU` (`0xA4`), `VK_RMENU` (`0xA5`), dan `VK_ESCAPE` (`0x1B`) langsung ditelan (`return 1`) pada baris pertama eksekusi callback saat `WM_KEYDOWN` dan `WM_SYSKEYDOWN`. Ini mematikan inisiasi `Alt+Tab`, `Alt+Esc`, dan Start Menu secara instan tanpa perlu reboot sistem!
  - **Preservasi Tombol Ctrl:** Tombol `Ctrl` (`0x11`, `0xA2`, `0xA3`) sengaja dibiarkan aktif saat ditekan sendiri agar shortcut editor C++ (`Ctrl+C`, `Ctrl+V`, `Ctrl+Z`, `Ctrl+S`, `Ctrl+A`) tetap berfungsi penuh di IDE CodeMirror 6.
  - Saat tombol dilepas (`WM_KEYUP`), event diteruskan ke `CallNextHookEx` agar Windows OS tahu tombol sudah dilepas, mencegah OS masuk ke kondisi stuck modifier.
- **Safe Message Pump Watchdog (Non-Destructive Thread Ping):**
  - Watchdog background memantau ketersediaan thread message pump secara berkala via `PostThreadMessageW(thread_id, WM_NULL = 0)`.
  - Loop pesan hook `GetMessageW` memperbarui heartbeat tanpa menyuntikkan keystroke sintetis, mencegah race condition dan menjamin hook terpasang 100% permanen.
- **Daftar Pemblokiran Mutlak (Return 1):**
  - **Tombol Windows:** `VK_LWIN` & `VK_RWIN` dicegat total (Start Menu mati mutlak).
  - **Kombinasi Tombol Windows:** Semua tombol saat `Win` aktif dicegat (`Win+A`, `Win+P`, `Win+L`, `Win+"+"` / `Win+"-"` Magnifier, `Win+Tab`, `Win+Shift+S`, `Win+D`, `Win+R`, `Win+S`, `Win+E`, `Win+X`, `Win+I`, `Win+G`, `Ctrl+Win+Left/Right/D/F4`).
  - **Navigasi Alt & Escape (Quadruple-Shield Defense):** `Alt+Tab`, `Alt+Shift+Tab`, `Ctrl+Alt+Tab`, `Alt+Esc`, `Alt+Space`, `Alt+F4`. Menggunakan empat lapisan pengamanan: (1) Penelanan instan tombol Alt (`0x12`, `0xA4`, `0xA5`) dan Esc (`0x1B`) di baris pertama callback hook, (2) Async hardware state polling (`is_alt_async = GetAsyncKeyState(0x12) | GetAsyncKeyState(0xA4) | GetAsyncKeyState(0xA5)`), (3) Hardware scancode fallback (`scan_code == 0x0F`), dan (4) OS-level Shell Hotkey Hijacking via `RegisterHotKey` (`MOD_ALT | MOD_NOREPEAT`, `VK_TAB`, `VK_ESCAPE`, dll).
  - **Navigasi Ctrl:** `Ctrl+Esc`, `Ctrl+Shift+Esc`, `Ctrl+Alt+Tab`, `Ctrl+P` (Print).
  - **Tangkapan Layar:** `VK_SNAPSHOT` (`PrintScreen` / `PrtScn`).

### 2.3 Layer 2: Penonaktifan & Penyembunyian Taskbar Windows
Untuk mencegah peserta mengklik tombol Start atau ikon Task View (Multi-Desktop):
- Win32 API `ShowWindow(SW_HIDE)` dan `EnableWindow(0)` dipanggil terhadap window taskbar utama (`Shell_TrayWnd`) dan taskbar multi-monitor (`Shell_SecondaryTrayWnd`).
- Saat Proctor keluar via PIN sah, taskbar otomatis dipulihkan ke `SW_SHOW` dan `EnableWindow(1)`.

### 2.4 Layer 3: Kebijakan Registry Windows Enterprise, Driver Scancode Map 40-Byte & WTS Session Guard
Untuk mencegah peserta membypass kiosk via task switcher OS, task manager, maupun lock screen:
- **Kebijakan Registry yang Diterapkan & Verifikasi Baca-Ulang:**
  - `HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System` -> `DisableTaskMgr = 1` (Menonaktifkan Task Manager dari menu Ctrl+Shift+Esc dan Ctrl+Alt+Del).
  - `HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System` -> `DisableLockWorkstation = 1` (Menonaktifkan penguncian komputer via Win+L).
  - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` -> `DisableTaskMgr = 1` & `DisableLockWorkstation = 1` (Jika dijalankan dengan hak Administrator).
  - `HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer` -> `NoLogoff = 1` & `NoWinKeys = 1`.
  - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer` -> `NoWinKeys = 1` (Jika dijalankan dengan hak Administrator).
  - `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced` -> `DisabledHotkeys = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"`.
  - `HKLM\SYSTEM\CurrentControlSet\Control\Keyboard Layout` -> `Scancode Map` binary 40-byte (Memetakan Left Win `E05B`, Right Win `E05C`, Left Alt `0038`, Right Alt `E038`, Escape `0001`, dan Apps `E05D` ke Null di level driver kernel keyboard NT jika dijalankan dengan hak Administrator; tombol Ctrl tetap dipertahankan).
  - Nilai `DisableLockWorkstation` dibaca ulang langsung setelah penulisan untuk memverifikasi apakah registry berhasil diterapkan di HKCU dan HKLM.
- **Deteksi Penguncian Sesi (WTSRegisterSessionNotification Guard):**
  - Pada Windows 11, `Win+L` dikelola oleh kernel Secure Attention Sequence (SAS). Jika sistem dijalankan tanpa hak administrator penuh, penulisan ke HKLM tidak diizinkan sehingga Win+L dapat memicu lock screen.
  - Sebagai mitigasi definitif, sistem memuat `WTSRegisterSessionNotification` secara dinamis dari `wtsapi32.dll` dan mendaftarkan window pesan tersembunyi (`HWND_MESSAGE`).
  - Saat Windows menerima event `WTS_SESSION_LOCK` (`WM_WTSSESSION_CHANGE`, code `0x7`), aplikasi segera:
    1. Mengaktifkan flag `IS_SESSION_LOCKED = true`.
    2. Menambah counter `session_lock_violations`.
    3. Mengirim payload telemetri darurat `FLAG_VIOLATION` ke proctor monitor server (`POST /api/telemetry`).
  - Saat sesi dibuka kembali (`WTS_SESSION_UNLOCK`, code `0x8`), sistem mencatat pemulihan dan mematikan flag sesi terkunci.
- **Catatan Kernel SAS (Win+L & Task Manager) & Script Administrator:**
  - Di Windows 11, `Win+L` dan `Ctrl+Alt+Del` dikelola langsung oleh kernel OS (Secure Attention Sequence / Winlogon) dan kebal terhadap seluruh user-space hook.
  - Untuk mematikan `Win+L` dan `Task Manager` secara tuntas di lab ujian, aplikasi Save Exam UPC harus dijalankan dengan hak Administrator via `bun run tauri:prod:admin` atau `scripts\run-kiosk-admin.bat`. Saat mendeteksi hak admin (`is_running_as_admin()`), backend Rust menulis ke `HKLM` dan `HKCU` sehingga kernel Windows menolak penguncian desktop.
- **Mekanisme Backup Atomik & Preservasi Sesi:**
  - Sebelum menulis kebijakan, nilai asli dibaca dan disimpan ke `%LOCALAPPDATA%\SaveExamUPC\registry_backup.json` menggunakan penulisan atomik (`.tmp` lalu `rename`).
  - Nilai yang sebelumnya tidak ada dicatat sebagai `null`.
  - Jika berkas backup yang belum dipulihkan (`"restored": false`) telah ada dari sesi sebelumnya, berkas backup **TIDAK DITIMPA** demi mencegah hilangnya nilai awal pengguna.
  - Saat Proctor keluar dengan PIN sah, kebijakan dipulihkan: nilai `null` dihapus dengan `RegDeleteValueW`, nilai awal dikembalikan, dan berkas ditandai `"restored": true`.
- **Catatan Efektivitas Explorer:**
  - Beberapa kebijakan Explorer (`NoLogoff`, `NoWinKeys`) memerlukan restart proses `explorer.exe` untuk diterapkan secara penuh oleh shell OS. Oleh karena itu, **Low-Level Keyboard Hook (`WH_KEYBOARD_LL`) bertindak sebagai benteng pertahanan instan dan mutlak utama** yang mencegat penekanan tombol secara real-time.

### 2.5 Startup Crash Recovery & Desktop 1-Click Recovery Folder
Jika komputer mati lampu, aplikasi mengalami crash, atau proses di-force terminate saat ujian berlangsung:
- **Pembersihan Otomatis saat Startup:**
  - Pada saat aplikasi diluncurkan kembali (`lib.rs setup`), sistem otomatis memanggil `check_and_recover_registry()`.
  - Jika ditemukan status `"restored": false`, sistem langsung memulihkan registry ke nilai awal pengguna, memulihkan Taskbar Windows (`Shell_TrayWnd` & `Shell_SecondaryTrayWnd`), melepaskan seluruh tombol modifier yang mungkin tersangkut via `release_all_modifier_keys()`, dan me-restart `explorer.exe`.
  - Jika berkas backup JSON tidak sengaja hilang, sistem menjalankan safety fallback yang langsung menghapus entri kebijakan kiosk di `HKCU` dan `HKLM`.
- **Rust Panic Hook Otomatis:**
  - `std::panic::set_hook` terpasang di awal eksekusi backend. Jika runtime mengalami unhandled panic saat kiosk aktif, panic hook secara otomatis memanggil `deactivate_kiosk_lockdown()`, mengembalikan registry, unhook keyboard, dan me-restart `explorer.exe` sebelum proses berhenti.
- **Folder Pemulihan & Penonaktifan Tombol di Desktop (`Desktop\Kiosk_Registry_Recovery`):**
  - Untuk kenyamanan panitia dan pengawas di lapangan, sistem secara otomatis menaruh toolkit di Desktop:
    1. `DISABLE_KEYS_REGISTRY.reg`: Skrip registry untuk mematikan tombol Left/Right Alt, Left/Right Win, Esc, dan Apps di level kernel driver Scancode Map.
    2. `APPLY_DISABLE_KEYS.bat`: Skrip batch auto-elevate RunAs Admin untuk menerapkan registry Scancode Map dengan satu klik.
    3. `DOUBLE_CLICK_TO_RESTORE.bat`: Skrip otomatis (auto-elevate Admin) yang mengimpor `.reg`, membunuh sisa modifier macet, me-refresh paksa `explorer.exe` untuk menghapus cache policy lama seketika, dan menyajikan opsi reboot otomatis jika ingin memulihkan driver kernel hardware keyboard 100%.
    4. `RELEASE_STUCK_KEYS.bat`: Skrip cepat untuk melepaskan modifier Windows/Ctrl/Alt/Shift yang tersangkut.
    5. `RESTORE_REGISTRY_NORMAL.reg`: Menghapus entri `DisableTaskMgr`, `DisableLockWorkstation`, `NoLogoff`, `NoWinKeys`, `DisabledHotkeys`, dan `Scancode Map`.
    6. `README_PETUNJUK.txt`: Panduan ringkas berbahasa Indonesia.
- **Pelepasan Tombol Modifier (Anti Stuck Keys):**
  - Callback `WH_KEYBOARD_LL` hanya menelan `VK_LWIN` & `VK_RWIN` pada `WM_KEYDOWN` dan **selalu meneruskan `WM_KEYUP` ke OS**, sehingga OS tidak pernah terjebak dalam kondisi tombol Win tertahan.
  - Ditambah pemanggilan sintetis `release_all_modifier_keys()` pada aktivasi, deaktivasi, pergantian fokus, dan pemulihan crash.

### 2.6 Background Process Watchdog (SEB Architecture)
Untuk menangani aplikasi pihak ketiga yang berjalan di latar belakang:
- Dedicated background worker di Rust berjalan setiap **1500 ms (1.5 detik)**.
- Menggunakan Win32 API snapshot proses `CreateToolhelp32Snapshot` (`TH32CS_SNAPPROCESS`).
- **Whitelist Ketat:**
  - PID aplikasi sendiri (`std::process::id()`).
  - Child process dari aplikasi sendiri (misal `g++.exe`, executable solusi peserta).
  - Executable inti aman: `msedgewebview2.exe` (WebView2 engine), `save-exam-upc.exe`, `save-exam-domjudge.exe`, `exam-app.exe`, `conhost.exe`.
- **Daftar Forbidden Processes (`contest.config.json`):**
  - Shortcut Flyout & Utility Bypass: `DisplaySwitch.exe` (Project Display / Win+P), `Magnify.exe` (Windows Magnifier / Win++).
  - Snipping Tool & Screen Clipping: `SnippingTool.exe`, `ScreenClippingHost.exe`, `ShareX.exe`, `Greenshot.exe`, `Lightshot.exe`, `ScreenToGif.exe`.
  - Screen Recording & Streaming: `obs64.exe`, `obs32.exe`, `Bandicam.exe`, `bdcam.exe`, `CamtasiaStudio.exe`, `GameBar.exe`, `GameBarFTServer.exe`.
  - Meeting & Remote Desktop: `Zoom.exe`, `Teams.exe`, `remoting_host.exe`, `AnyDesk.exe`, `TeamViewer.exe`, `RustDesk.exe`, `Discord.exe`, `vncserver.exe`, `mstsc.exe`.
- **Terminasi & Telemetri:**
  - Mematikan proses terlarang langsung via Win32 API `OpenProcess(PROCESS_TERMINATE, ...)` -> `TerminateProcess()`.
  - Jika gagal akibat elevation, fallback otomatis ke `taskkill /F /PID` dengan flag `CREATE_NO_WINDOW` (`0x08000000`) agar tidak memunculkan cmd popup.
  - Log peringatan dicatat dan event pelanggaran dikirim secara asinkron ke server monitoring jika dikonfigurasi (`POST /api/telemetry`).

### 2.7 Pre-Check Aplikasi Eksternal Berbasis Whitelist & Clean Desktop Enforcer
Sistem menerapkan prinsip **Strict Whitelist** untuk memastikan desktop bersih (*clean desktop*) saat ujian berlangsung:
- **Deteksi Menyeluruh (Bukan Hanya Peramban):** Sistem memindai seluruh proses aktif pada sesi interaktif pengguna (`ProcessIdToSessionId`). Setiap proses yang berada di luar Whitelist aman (Save Exam, WebView2, MinGW toolchain, dan proses inti OS Windows) dikategorikan sebagai aplikasi yang tidak dibutuhkan (termasuk seluruh browser, IDE/editor eksternal, messaging/chat, office, recording/streaming, utilities, dan AI assistant).
- **Peringatan & Kontrol di Halaman Login:** Menampilkan banner peringatan komprehensif dengan daftar aplikasi yang terkelompokkan rapi, tombol **"Tutup Semua Aplikasi"** (mematikan seluruh aplikasi non-whitelist via Win32 `TerminateProcess` + `taskkill` fallback), tombol "Periksa Ulang", dan tombol "Hubungi Pengawas".
- **Silent Recheck Saat Memulai Ujian:** Saat tombol "Masuk & Verifikasi Akun" atau "Masuk Kontes Ini" ditekan, sistem melakukan verifikasi ulang otomatis. Jika masih ada aplikasi non-whitelist yang berjalan, akses diblokir seketika dengan modal dialog yang dilengkapi tombol instan untuk menutup seluruh aplikasi tersebut sebelum masuk arena.

### 2.8 Pengecualian Foreground Lock Saat Modal PIN Pengawas Aktif
- Watchdog foreground (80ms loop) secara otomatis dilewati (`skip refocus`) ketika flag atomic `IS_PROCTOR_MODAL_ACTIVE` bernilai `true`.
- Menjamin pengawas dapat mengetik PIN, berinteraksi dengan dialog bantuan, atau menutup ujian tanpa terganggu oleh perebutan fokus window.

### 2.9 Windows Clipboard Cleaner
- Saat Kiosk diaktifkan dan saat dideaktivasi, backend Rust memanggil Win32 API:
  ```rust
  OpenClipboard(null);
  EmptyClipboard();
  CloseClipboard();
  ```
- Mencegah peserta menempelkan (paste) teks/kode yang disalin sebelum aplikasi dibuka atau membocorkan kode setelah aplikasi ditutup.

### 2.10 Analisis Eksperimental Isolated Desktop (`CreateDesktopW`)
- Opsi `"isolated_desktop": false` dipertahankan di konfigurasi sebagai fitur eksperimental.
- **Catatan Arsitektur WebView2:** Engine WebView2 (Chromium multi-process) memerlukan penanganan token sesi dan WindowStation inheritance yang sangat kompleks jika dijalankan di desktop Win32 terpisah (`CreateDesktopW`), yang sering mengakibatkan render failure jika dijalankan dalam satu proses terpadu (kendala serupa yang dialami Safe Exam Browser saat migrasi CEF ke WebView2). Implementasi produksi masa depan memerlukan architecture launcher mandiri. Arsitektur berlapis saat ini (Default Desktop + Hidden Taskbar + Watchdogs + Registry Policies + Hook + ExcludeFromCapture) memberikan stabilitas 100% tanpa risiko crash WebView2.

### 2.11 Watchdog Foreground Agresif & Anti-Alt-Tab Snapback
- Dedicated background worker di Rust (interval 80ms) memantau `GetForegroundWindow()`.
- Jika window lain mencoba mengambil fokus atau muncul di atas:
  ```rust
  SetForegroundWindow(target);
  BringWindowToTop(target);
  SetWindowPos(target, HWND_TOPMOST, 0, 0, 0, 0, SWP_NOSIZE | SWP_NOMOVE | SWP_SHOWWINDOW);
  ```
- Save Exam UPC langsung dipaksa kembali ke posisi teratas seketika, mencegah peserta mengintip aplikasi lain.
- Frontend React melengkapi dengan listener `window.onblur` yang langsung memanggil `window.focus()`.

### 2.12 Anti-Screenshot Hardware OS (`SetWindowDisplayAffinity`)
Untuk mencegah perekaman layar, tangkapan layar, maupun sharing via software pihak ketiga (OBS, Discord, Snipping Tool, remote viewer):
- Backend Rust menerapkan proteksi hardware display affinity via Win32 FFI:
  ```rust
  SetWindowDisplayAffinity(hwnd, WDA_EXCLUDEFROMCAPTURE); // fallback ke WDA_MONITOR (1)
  ```
- Window Save Exam UPC otomatis di-render sebagai area hitam pekat (*black screen*) di semua tangkapan layar, screen recording, maupun screen sharing software.

### 2.13 Pencegahan Penutupan Window (`prevent_close`) & Proctor PIN
- Event OS `WindowEvent::CloseRequested` dicegat oleh Tauri window listener menggunakan `api.prevent_close()`.
- Tombol close dari taskbar OS, kombinasi penutup window, maupun sinyal quit tidak dapat mematikan window aplikasi.
- Tombol "Keluar" di UI hanya menampilkan modal PIN Pengawas (`ProctorModal.tsx`).
- Input PIN dikirim ke command Rust `verify_pin(pin: String)` yang mencocokkan hash SHA-256 terhadap `proctor_pin_hash` di `contest.config.json`.
- Aplikasi hanya dapat keluar secara sah melalui command `exit_kiosk` yang memanggil `deactivate_kiosk_lockdown()`, memulihkan taskbar, membersihkan registry, melepaskan hook, lalu memanggil `app_handle.exit(0)`.

### 2.8 Konfigurasi & Prosedur Penggantian PIN Pengawas (Strict Build-Time Baking)
- **Zero Runtime Bypass:** PIN pengawas **TIDAK DAPAT** di-override di komputer lab melalui berkas `.env` atau manipulasi teks runtime apa pun. Hal ini untuk mencegah peserta ujian membuat berkas `.env` sendiri di lab untuk membypass pengawasan.
- **Format Penyimpanan:** PIN disimpan dalam format **SHA-256 Hex Digest (64 karakter)** dan di-embed (baked) secara mutlak ke dalam executable `.exe` saat proses kompilasi melalui `include_str!("../../../contest.config.json")`.
- **PIN Default:** `123456`  
  - SHA-256: `8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92`
- **Cara Mengganti PIN Pengawas:**
  1. **Sebelum Build Lokal / Release:**
     Generate hash SHA-256 dari PIN baru via PowerShell:
     ```powershell
     $pin = "PIN_BARU_PENGAWAS"
     [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash([Text.Encoding]::UTF8.GetBytes($pin))).Replace("-","").ToLower()
     ```
     Lalu perbarui nilai `"proctor_pin_hash"` di `contest.config.json` sebelum menjalankan kompilasi.
  2. **Via GitHub Actions CI/CD (Workflow Dispatch / Secret):**
     Saat memicu build rilis di GitHub Actions (`Build Matrix & Release`):
     - Masukkan PIN baru pada input `proctor_pin` di form *Run workflow*, atau
     - Simpan PIN di GitHub Repository Secrets (`PROCTOR_PIN`).
     GitHub Actions otomatis menghitung hash SHA-256 dan menanamkannya ke dalam executable Windows release (`.exe`, `.msi`, `.zip`).

---

## 3. Ephemeral Statement Protection (Zero-Disk Leakage)

Untuk melindungi kerahasiaan soal kompetisi agar tidak tertinggal di hard disk komputer lab:
1. **RAM-Only Cache (`statement_ram_cache`):**
   - Berkas PDF soal yang diunduh dari endpoint DOMjudge (`GET /api/v4/contests/{cid}/problems/{id}/statement`) langsung dikonversi ke Base64 data URL (`data:application/pdf;base64,...`) di backend Rust.
   - Data Base64 ini disimpan secara eksklusif dalam memori RAM (`Arc<RwLock<HashMap<String, String>>>`).
   - Tidak ada satu pun byte berkas soal yang ditulis ke hard disk (seluruh direktori cache disk `.cache/statements/` dieliminasi total).
2. **Pemusnahan Otomatis (Instant Destruction):**
   - Saat aplikasi ditutup, force kill, crash, ataupun komputer di-restart/mati lampu, seluruh cache soal di RAM musnah 100%. Peserta tidak dapat menemukan jejak file soal di disk lab.
3. **Fitur Hard Reload Soal:**
   - Menyediakan tombol Hard Reload di viewer soal (`domjudge_get_statement` dengan `force_refresh: true`).
   - Jika admin/juri memperbarui deskripsi soal di DOMjudge saat kompetisi berjalan, peserta dapat langsung menarik revisi soal terbaru langsung ke memori RAM tanpa sisa berkas lama.

---

## 4. Scratchpad Isolation (TMP 1, 2, 3)

Peserta dibekali 3 berkas coret-coretan lokal (`scratch_1.cpp`, `scratch_2.cpp`, `scratch_3.cpp`) untuk eksplorasi rumus, hitung pola, dan eksperimen algoritma:
- **Tersimpan di Workspace:** Terletak di `$DESKTOP/Contest_Submissions/scratch_*.cpp`.
- **Kompilasi Lokal Aman:** Dapat dikompilasi (`F9`) dan dijalankan (`F11`) di bawah Process Sandbox C++.
- **Blokir Submit Mutlak:** Sistem frontend dan IPC secara tegas mematikan tombol submit (`canSubmit = false`). Berkas scratchpad tidak dapat disubmit ke server DOMjudge demi mencegah salah submit atau diskualifikasi.

---

## 5. Tauri v2 Capabilities & Least Privilege

Tauri v2 menggunakan model izin berbasis capabilities JSON (`src-tauri/capabilities/default.json`):

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Izin minimal untuk window exam",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "shell:allow-execute",
    "fs:allow-read-text-file",
    "fs:allow-write-text-file"
  ]
}
```

### 5.1 Pembatasan File System (`fs` Scope)
Akses pembacaan dan penulisan dibatasi mutlak hanya pada:
- `$DESKTOP/Contest_Submissions/*`
- Path traversal (seperti `../` atau symlink) dideteksi dan ditolak oleh backend Rust (`PathBuf::canonicalize()`).

### 5.2 Pembatasan Shell (`shell` Scope)
- Plugin shell hanya diizinkan mengeksekusi path biner compiler C++ (`g++.exe`) yang terdaftar di whitelist konfigurasi lomba.
- Eksekusi arbitrary shell script atau command line interpreter (`cmd.exe`, `powershell.exe`, `bash`) diblokir.

### 5.3 Content Security Policy (CSP)
```text
default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://upcvi.online; img-src 'self' data: https://upcvi.online; frame-src 'self' data:;
```
- Menolak koneksi jaringan ke alamat di luar domain DOMjudge yang tertera di `contest.config.json`.
- `frame-src 'self' data:;` mengizinkan rendering PDF viewer ephemeral dari Base64 memory URL secara aman.

---

## 6. Process Sandbox untuk Kode Peserta

Solusi C++ yang dikompilasi dijalankan di bawah lingkungan isolasi:

### 6.1 Batasan Windows (Job Objects)
Pada Windows, proses binary peserta dimasukkan ke dalam **Windows Job Object**:
- `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` — Otomatis membunuh child process jika proses induk mati.
- `JOB_OBJECT_LIMIT_PROCESS_MEMORY` — Membatasi memori maksimum (misal 512 MB). Jika alokasi melebihi batas, OS membangkitkan sinyal OOM/MLE.
- `JOB_OBJECT_LIMIT_PROCESS_TIME` — Membatasi CPU time (misal 5 detik).

### 6.2 Batasan Unix / Linux (`rlimit`)
Pada sistem Linux, backend Rust menyetel limit via `libc::setrlimit`:
- `RLIMIT_CPU` — Batas waktu CPU sebelum sinyal `SIGXCPU` dikirim.
- `RLIMIT_AS` (Address Space) — Batas memori virtual maksimum.
- `RLIMIT_FSIZE` — Batas ukuran output berkas maksimum (10 MB).

### 6.3 Isolasi Jaringan
Proses binary peserta tidak diberikan handle socket jaringan. Peserta tidak dapat melakukan koneksi internet lokal atau remote dari kode program mereka.

---

## 7. Keamanan DOMjudge Webview & Integrasi Session

- **Domain Whitelist:** Webview DOMjudge diisolasi hanya pada host yang ditentukan di `contest.config.json` (`https://upcvi.online`). Navigasi ke domain asing langsung dibatalkan via event navigation handler.
- **Kredensial:** Password akun peserta disimpan secara eksklusif dalam memori RAM backend (`Arc<Mutex<Option<Credentials>>>`) dan tidak pernah ditulis ke `localStorage` browser atau disk lokal.
- **Strict Anti-Mock:** Seluruh interaksi DOMjudge dan compiler berkomunikasi langsung dengan sistem nyata tanpa adanya mock data palsu.

---

## 8. Anti-Tamper & Proteksi Materi Ujian / Modul Referensi

Untuk mendukung modul referensi resmi (Modul Praktikum SDA dan Modul STRAGO UTS) tanpa membuka celah kecurangan atau penyelundupan catatan sontekan:
1. **Immutable Static Whitelist:** Backend Rust tidak pernah memindai sembarang berkas dari direktori disk. Hanya ID dan berkas yang terdaftar secara eksplisit di manifest statis kode Rust (`sda`, `strago`) yang dapat dilayani oleh IPC `materials_get_pdf`.
2. **Kriptografi SHA-256 Checksum Enforcement:** Setiap payload materi wajib lolos verifikasi hash SHA-256 yang di-hardcode di biner backend. Jika berkas diubah 1 byte saja oleh peserta atau hacker lab, sistem langsung menolak dan membangkitkan `AppError::Security("Integrity check failed")`.
3. **Embedded Binary Payload Immunity:** Payload byte PDF di-embed langsung ke dalam biner executable (`include_bytes!`) atau bundle terisolasi, sehingga sistem kebal terhadap upaya replace, injeksi berkas palsu, atau penghapusan berkas di hard disk komputer laboratorium.
4. **Anti-Leak Display Protection:** Modul materi dirender via HTML5 canvas memori RAM tanpa tombol unduh/simpan, bebas dari dialog cetak (`@media print { display: none }`), dan terlindungi oleh proteksi anti-capture hardware OS (`WDA_EXCLUDEFROMCAPTURE`).

