# Product Requirement Document (PRD)

## 1. Overview & Goal
Fitur **Focus Blocker & Screen Takeover** pada Renotch di macOS bertujuan untuk memblokir distraksi website (seperti Threads, Instagram, YouTube) tanpa menyentuh layer network/DNS. Fitur ini bekerja dengan memantau *active window title* via macOS Accessibility API dan secara instan menganimasikan notch menjadi overlay layar penuh (*fullscreen takeover*) yang merender halaman HTML custom lokal.

---

## 2. Problem Statement
* Pendekatan network-level (`/etc/hosts`, PAC file, local proxy) sering gagal karena browser modern memakai DNS-over-HTTPS (DoH).
* Mengarahkan HTTPS ke localhost memicu error SSL (`NET::ERR_CERT_COMMON_NAME_INVALID`).
* User membutuhkan feedback visual yang jelas dan mulus saat mengakses situs terlarang tanpa merusak konfigurasi browser.

---

## 3. Core Architecture & Flow

[Background Polling / AX Observer (200-300ms)]
│
▼
[Deteksi Frontmost App + Window Title / URL Bar]
│
Match Blacklist Pattern?
├── NO  ──> Notch tetap di state normal
└── YES ──> Trigger Fullscreen Overlay Hijack
│
▼
[Animasi Notch Expand ke Fullscreen]
│
▼
[WKWebView memuat blocked.html lokal]
│
▼
[User Action: "Kembali Kerja" / Tutup Tab]


---

## 4. Functional Requirements

### 4.1 Window & Title Monitoring
* **Target Browsers:** Google Chrome, Safari, Brave, Arc, Microsoft Edge.
* **Mechanism:** Menggunakan `NSWorkspace.shared.frontmostApplication` dikombinasikan dengan `AXUIElementCopyAttributeValue` (`kAXTitleAttribute` atau `kAXValueAttribute` pada URL text field).
* **Interval:** Event-driven via `NSWorkspace.didActivateApplicationNotification` + Polling timer fallback setiap 250ms.

### 4.2 Pattern Matching Engine
* Menyediakan daftar default domain/keywords:
  * `threads.net`, `instagram.com`, `twitter.com`, `x.com`, `youtube.com`
* Regex case-insensitive matching terhadap judul window atau URL string.
* Konfigurasi daftar blacklist tersimpan di local storage (JSON / User Defaults).

### 4.3 Overlay & UI Transition
* **Window Level:** `NSWindow.Level.floating` atau `NSWindow.Level.screenSaver` (selalu di atas aplikasi lain).
* **Animation:** Transisi ukuran dari status compact notch (`CGSize(width: 220, height: 35)`) membesar secara elastis (*spring animation*) hingga `NSScreen.main.frame`.
* **Rendering Engine:** Memakai `WKWebView` untuk memuat template file HTML custom lokal (`file:///.../blocked.html`).

### 4.4 User Actions inside HTML Overlay
* **Tombol "Tutup Tab & Kembali Kerja":**
  * Renotch mengeksekusi AppleScript untuk menutup tab aktif di browser:
    ```applescript
    tell application "Google Chrome" to close active tab of front window
    ```
  * Notch otomatis mengecil (*collapse*) kembali ke ukuran semula.
* **Tombol "Bypass / Istirahat 5 Menit":**
  * Memberikan jeda sementara (*cooldown timer*) sebelum memicu blocker lagi.

---

## 5. Non-Functional & Technical Requirements

* **Permissions:** Wajib meminta izin **macOS Accessibility** (`AXIsProcessTrustedWithOptions`). Jika belum aktif, tampilkan prompt onboarding.
* **Resource Usage:** CPU usage background watcher harus di bawah 1% saat idle.
* **Zero Network Interference:** Tidak mengubah file `/etc/hosts`, tidak mengaktifkan sistem proxy, dan tidak membutuhkan sertifikat root CA.
* **Offline Compatible:** File `blocked.html` dan seluruh aset visual (CSS/JS/gambar) disimpan secara lokal di dalam bundle aplikasi.

---

## 6. Success Metrics
* 0% issue terkait sertifikat SSL / DoH bypass.
* Response time dari user membuka tab hingga layar tertutup < 350ms.
* Animasi transisi notch ke fullscreen berjalan mulus pada 60/120 Hz (ProMotion display).