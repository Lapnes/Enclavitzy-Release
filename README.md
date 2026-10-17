<div align="center">

# 🛡️ Enclavitzy

### *Secure National Programming Contest Exam Browser & Native C++ IDE*

A next-generation, high-performance desktop kiosk lockdown browser and native C++23 programming contest environment engineered for **DOMjudge** on-site competitions.

[![Tauri v2](https://img.shields.io/badge/Tauri-v2.1-24C8DB?style=for-the-badge&logo=tauri&logoColor=white)](https://tauri.app)
[![Rust](https://img.shields.io/badge/Rust-1.77+-orange?style=for-the-badge&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![React 18](https://img.shields.io/badge/React-18.3-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-Strict-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![MinGW GCC](https://img.shields.io/badge/GCC-13.2%20(C%2B%2B23)-00599C?style=for-the-badge&logo=cplusplus&logoColor=white)](https://gcc.gnu.org)
[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%2F%2011%20(x64)-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://microsoft.com/windows)

<br/>

**⚡ < 35 MB RAM Idle &bull; 🚀 < 0.8s Cold Startup &bull; 📦 Ultra-Lightweight &bull; 🔒 Kernel Win32 Lockdown &bull; 📡 Real-time Proctor Telemetry**

<br/>

[📥 Download Releases](https://github.com/Lapnes/Enclavitzy-Release/releases) •
[Features](#-key-features) •
[Package Formats](#-available-package-formats) •
[Organizer Guide](#-contest-organizer-guide) •
[Documentation](#-documentation)

---

</div>

<br/>

## 🌟 Overview

**Enclavitzy** is purpose-built for university and national-level Competitive Programming contests (ICPC, OSN, National Olympiads) running on **DOMjudge**. Traditional exam browsers are either bloated Electron wrappers that consume 400+ MB of RAM on lab machines or rigid WebViews that lack native C++ compilers, sample test runners, and real-time proctor surveillance.

Enclavitzy unifies **hardware-level desktop lockdown**, a **lightweight CodeMirror 6 C++ IDE with VS Code-style minimap**, **native MinGW GCC compilation**, **RAM-only PDF problem statement streaming**, and a **standalone Proctor Telemetry WebSocket server** into a single, cohesive executable.

---

## 📥 Available Package Formats

Download the latest version directly from the **[GitHub Releases Page](https://github.com/Lapnes/Enclavitzy-Release/releases)**.

| Varian | Format Berkas | Keterangan | Rekomendasi Penggunaan |
|:---|:---|:---|:---|
| **Full (Bundled GCC)** | `.exe` (NSIS Setup) | Installer dengan compiler MinGW GCC 13.2 ter-bundling | Lab komputer tanpa GCC sistem (Siap pakai) |
| **Full (Bundled GCC)** | `.msi` (Enterprise MSI) | Installer Windows standar deployment GPO/Active Directory | Distribusi terpusat lab via Admin Server |
| **Full (Bundled GCC)** | `.zip` (Portable Standalone) | Executable siap pakai tanpa perlu proses instalasi | Flashdisk pengawas / lab tanpa hak instalasi |
| **Vanilla** | `.exe` (NSIS Setup) | Installer ultra-ringan (~4 MB), memanfaatkan compiler lokal | Lab yang sudah memiliki MinGW / Code::Blocks |
| **Vanilla** | `.msi` (Enterprise MSI) | Installer MSI ultra-ringan (~6 MB) | Lab dengan MinGW di PATH |
| **Vanilla** | `.zip` (Portable Standalone) | Portable zip ultra-ringan (~6 MB) tanpa instalasi | Praktis untuk peserta dengan compiler lokal |
 
> 💡 **Panduan Silent Installation (Automasi Deploy Lab Komputer):**
> - **MSI (.msi) — CMD / Batch / GPO:** `msiexec /i "Enclavitzy-v0.1.1-windows-x64-full.msi" /quiet /qn /norestart`
> - **NSIS (.exe) — CMD / Batch:** `Enclavitzy-v0.1.1-windows-x64-full-setup.exe /S`
> - **PowerShell Remote:** `Start-Process msiexec.exe -ArgumentList '/i "Enclavitzy-v0.1.1-windows-x64-full.msi" /quiet /qn /norestart' -Wait`

---

## ✨ Key Features

### 1. 🔒 Enterprise Kiosk Lockdown & Anti-Cheat
- **Otomatisasi Hak Administrator (UAC Manifest):** Executable Enclavitzy menyematkan Windows Application Manifest resmi (`requestedExecutionLevel level="requireAdministrator"`). Windows secara otomatis memunculkan prompt Administrator saat aplikasi dibuka tanpa perlu klik kanan manual.
- **Eliminasi Total Layar SAS (`Ctrl+Alt+Del`):** Menutup celah peserta keluar dari kiosk via logout Windows. Dengan privilese Administrator, Enclavitzy menerapkan Group Policy HKLM & HKCU yang meniadakan tombol Sign Out (`NoLogoff = 1`), Switch User (`HideFastUserSwitching = 1`), Change Password (`DisableChangePassword = 1`), Lock (`DisableLockWorkstation = 1`), Task Manager (`DisableTaskMgr = 1`), dan Power Shutdown (`NoClose = 1`).
- **Zero Window Switch:** Memblokir tombol Windows (`WinKey`), `Alt+Tab`, `Alt+Esc`, `Alt+F4`, `Ctrl+Esc`, dan `PrintScreen` via Win32 Low-Level Keyboard Hook (`WH_KEYBOARD_LL`).
- **Selective Shortcut Whitelist:** Mempertahankan pintasan C++ penting seperti `Alt+ArrowUp`/`Down` (move line) dan `Shift+Alt+F` (format kode), sembari menelan seluruh kombinasi navigasi berbahaya.
- **Hardware Blackout Anti-Capture:** Memanfaatkan API Windows Kernel `SetWindowDisplayAffinity(WDA_EXCLUDEFROMCAPTURE)`. Layar akan otomatis hitam pekat saat direkam OBS, Discord, Snipping Tool, atau capture card.
- **Start Menu & Explorer Suppression:** Otomatis mematikan `explorer.exe`, `SearchHost.exe`, dan Task Manager saat ujian aktif demi menjamin lingkungan ujian bersih.
- **1-Click Recovery Folder:** Secara otomatis membuat folder pemulihan di `Desktop/Enclavitzy/Kiosk_Registry_Recovery/` lengkap dengan script pemulihan registry satu-klik jika terjadi force shutdown.

### 2. 🧠 Ephemeral Zero-Disk Statement Cache
- Berkas soal PDF (`GET /api/v4/contests/{cid}/problems/{id}/statement`) diunduh langsung ke **RAM backend Rust**, di-decode secara *in-memory*, dan dirender via HTML5 Canvas (`pdfjs-dist`).
- **Zero Disk Leakage:** Tidak ada file PDF soal yang tersimpan di harddisk lab, mencegah kebocoran soal kompetisi nasional di komputer bersama.

### 3. 💻 Native C++23 IDE & Integrated Test Runner
- Editor ringan berbasis **CodeMirror 6** (~140 KB gzipped) dengan tema dark contest tajam, VS Code Minimap canvas, dan indentasi cerdas 4 spasi.
- **Magic Wand Auto-Formatter (`Shift+Alt+F`):** Merapikan sintaks C++, kurung kurawal, dan operator biner dalam 0ms tanpa dependensi WASM/LLVM eksternal.
- **Integrated Samples Runner:** Otomatis mengunduh test case contoh dari DOMjudge dan mengujinya secara lokal dengan timeout aman 21 detik dan tombol merah **Force STOP**.
- **Contest-Scoped Submissions:** Seluruh solusi tersimpan rapi dan terisolasi per kontes di `Desktop/Enclavitzy/Submissions/{hash}_{contest_name}/` dengan penamaan otomatis `{ProblemLabel}_{ProblemName}.cpp`.

### 4. 📡 Standalone Proctor Telemetry Monitoring
- Client secara asinkron mendispatch telemetri tombol kombinasi mencurigakan (`DANGER`, `WARNING`, `SUS`) ke Proctor Monitor Server (`:3001`).
- Pengawas dapat memantau seluruh peserta ujian secara real-time via dashboard web interaktif dengan filter tim dan notifikasi audio.

### 5. 🔋 Real-Time Hardware Battery Monitor
- Membaca status daya hardware secara native via Win32 API `GetSystemPowerStatus` (< 0.001ms, read-only ACPI register).
- Menampilkan status daya AC untuk PC desktop, dan persentase baterai serta status pengisian daya untuk laptop dengan peringatan visual berkedip saat baterai $\le 20\%$.
- 100% aman dan tidak melonggarkan keamanan kiosk.

### 6. ⏱️ Real-Time Contest Clock & Countdown Timer HUD (Center Header Pill)
- Disematkan di bagian tengah navigasi atas (Top Header Bar) dalam bentuk Center HUD Pill yang bersih dan hemat ruang.
- Waktu disinkronkan presisi terhadap header HTTP `Date` server DOMjudge (`server_time_offset_secs`) untuk mencegah manipulasi jam lokal PC peserta.
- Format angka monospace tabular (`font-mono tabular-nums`) anti-getar (*zero layout jitter*), dengan transisi dinamis (Upcoming, Running, Warning $\le 15$ menit berkedip, dan Ended).

---

## 📋 Contest Organizer Guide

Untuk panitia atau pengawas kontes, konfigurasi dasar dapat disesuaikan pada file [`contest.config.json`](./contest.config.json):

```json
{
  "app_name": "Enclavitzy",
  "domjudge_url": "https://upcvi.online",
  "proctor_pin_hash": "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92",
  "compiler": {
    "flags": ["-O2", "-std=c++23", "-Wall", "-Wextra"],
    "compile_timeout_secs": 30,
    "run_timeout_secs": 21,
    "max_output_kb": 1024
  },
  "kiosk": {
    "enabled_in_production": true,
    "block_shortcuts": true,
    "disable_explorer_shell": true
  }
}
```

- **`domjudge_url`**: Arahkan ke domain server DOMjudge Anda (misal: `https://domjudge.kampus.ac.id`).
- **`proctor_pin_hash`**: Hash SHA-256 dari PIN pengawas untuk keluar dari mode Kiosk (Default: PIN `123456`).
- **`compiler.flags`**: Opsi kompilasi GCC lokal (standar ICPC/IOI: `-O2 -std=c++23`).

---

## 📚 Documentation

Dokumentasi arsitektur, keamanan, dan integrasi API lengkap:
- 🏛️ **[docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)** — Arsitektur sistem 2-tier (Rust backend + React frontend), lifecycle kontes, dan model manajemen memori.
- 🔐 **[docs/SECURITY.md](./docs/SECURITY.md)** — Model keamanan 4 lapis, Win32 hook lockdown, anti-tamper materi, dan registry recovery.
- 🌐 **[docs/API_DOMJUDGE.md](./docs/API_DOMJUDGE.md)** — Spesifikasi empiris integrasi REST API v4 DOMjudge untuk peserta lomba.

---

<div align="center">

**Enclavitzy** &bull; *Engineered for Integrity, Performance, and Precision in Competitive Programming.*

</div>
