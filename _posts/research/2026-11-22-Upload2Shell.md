---
title: Upload2Shell - RCE Ghostscript Lewat Upload File Biasa
author: Thyrh07
date: 2026-11-22 08:00:00 +0700
categories: [Research]
tags: [Upload2Shell, Cve, Rce, Ghostscript, Vulnerability]
excerpt: ""
---

>   **You are solely responsible for any actions or activities related to the material on this website. Some of the content here may be harmful or dangerous if misused. If you do not fully understand any material on this site, please leave. Before accessing or using these materials in any way, review the laws that apply in your province or country. All materials are provided for educational and research purposes only.**


## **<span style='color:#ff5555'>Ringkasan</span>**
***

Upload2Shell adalah kerentanan Ghostscript yang memungkinkan remote code execution (RCE) lewat file yang di-upload, dikirim lewat email, atau di-print — tanpa perlu login di beberapa jalur yang sudah dibuktikan peneliti. File itu tidak perlu bernama `.ps` atau `.pdf`: file bernama `.png` atau `.jpg` yang berisi PostScript/EPS tetap bisa sampai ke Ghostscript lewat proses preview, thumbnail, atau konversi di belakang layar.

- **CVE:** CVE-2026-125396 (CNA: VulnCheck)
- **Nama:** Upload2Shell
- **Jenis:** Use-after-free (CWE-416) di Type 1 font loader Ghostscript, berujung sandbox escape `-dSAFER`
- **Skor:** CVSS 4.0 **8.5 (High)** — `CVSS:4.0/AV:L/AC:L/AT:N/PR:N/UI:P/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N`
- **Penemu:** Paulos Yibelo, pwn.ai — riset dipublikasikan 17 November 2026
- **Pemicu utama:** Upload file, lampiran email yang di-thumbnail, atau print job ke shared queue — aplikasi yang bersangkutan tidak perlu memanggil Ghostscript secara langsung
- **Terdampak:** Ghostscript sebelum **10.08.0**
- **Status eksploitasi:** Belum ada laporan eksploitasi di alam liar per 22 November 2026; belum masuk CISA KEV

> ⚠ **Klarifikasi penting:** Skor CVSS-nya tercatat `AV:L` (local) karena yang dinilai adalah celah di interpreter Ghostscript itu sendiri — bukan karena serangannya harus lokal. Dalam pengujian pwn.ai, file itu sampai ke Ghostscript dari jaringan: lewat upload web, email, dan print job. Jangan membaca skor itu sebagai "harus sudah punya akses ke server". Sebaliknya, ini juga bukan satu exploit universal: tiap aplikasi punya syarat konfigurasi sendiri (lihat tabel jalur di bawah), dan perintah yang jalan adalah sebagai akun proses rendering — bukan otomatis root.

## **<span style='color:#ff5555'>Apa Itu Upload2Shell</span>**
***

Ghostscript adalah interpreter yang dipakai banyak server Linux untuk merender PDF dan PostScript — dan, yang lebih penting, ia duduk di belakang library yang jauh lebih umum: ImageMagick, Pillow, PHP Imagick, dan sebagian build libvips. Saat aplikasi membuat preview, thumbnail, atau mengonversi gambar, library itu bisa menyerahkan konten PostScript/EPS ke Ghostscript, bahkan kalau file yang di-upload bernama `.png` atau `.jpg`.

Bug-nya ada di bagian Ghostscript yang memuat font Type 1 — dan sebagian implementasinya ditulis dalam PostScript itu sendiri. Prosedur `.putgstringcopy` di `gs_fonts.ps` memanggil operator privileged `.forceput` tanpa memeriksa dulu tipe operand-nya. Dokumen jahat sengaja memicu type error di dalam prosedur itu, lalu menangkap error tersebut dengan error handler-nya sendiri — dan dari handler itu ia mendapatkan kembali `.forceput` yang bisa dipakai.

Dari situ, `.forceput` dipakai untuk membiarkan referensi ke objek lokal tetap hidup setelah `restore` membebaskan memorinya: use-after-free. Memori yang dipakai ulang itu memberi penyerang primitive write-what-where, yang dipakai untuk menulis ulang state yang menegakkan `-dSAFER` — sandbox Ghostscript yang seharusnya melarang dokumen menjalankan perintah atau mengakses file. Setelah sandbox itu dimatikan, perintah jalan sebagai akun proses rendering.

Dua detail yang membuat celah ini praktis: exploit-nya membawa font Type 1-nya sendiri, jadi tidak butuh font tertentu terpasang di server; dan fix upstream memindahkan helper yang rentan itu ke kode C supaya tidak lagi bergantung pada `.forceput` (commit `1484854e`, Artifex Bug 709684, dikreditkan ke Ken Sharp).

## **<span style='color:#ff5555'>Jalur yang Sudah Dibuktikan Peneliti</span>**
***

pwn.ai menguji sekitar sepuluh aplikasi dan mencatat eksekusi perintah di masing-masing lewat fitur sehari-hari. Yang sudah dibuktikan, per 22 November 2026:

| Aplikasi / layanan | Jalur | Login? | Akun proses |
|----------|-------|--------|-------------|
| WordPress core (via Imagick / ImageMagick) | Media preview | Perlu akun yang bisa upload file (Author ke atas) | Web server |
| WordPress plugin Forminator & Essential Addons for Elementor | Form upload publik | Tidak, di konfigurasi yang diuji | Web server |
| Roundcube 1.7.2 | Lampiran email di-thumbnail saat pesan dibuka | Pengirim tidak butuh akun; penerima cukup membuka pesan, tanpa klik lampiran | `www-data` |
| CUPS, shared queue di Debian 13 | Print job IPP anonim | Tidak, tanpa kredensial | `lp` |
| Plesk Obsidian | Roundcube yang dibundel Plesk | Tergantung konfigurasi | Web server |
| MediaWiki | Upload PDF anonim via PdfHandler | Tidak, di konfigurasi yang diuji | Web server |
| DokuWiki, Mayan EDMS, ResourceSpace | Upload / pemrosesan dokumen | Tergantung konfigurasi | Web server |
| Google document renderer | "Production hit" di layanan pemrosesan dokumen Google | — | — |

Catatan untuk tiga jalur di atas:

- **WordPress core** punya CVE sendiri untuk jalur upload-nya, **CVE-2026-65640**, dan sudah diperbaiki di WordPress **7.0.4 (12 Agustus 2026)**, di-backport termasuk ke 6.9.7. Jadi patch WordPress menutup jalur itu, tapi tidak memperbaiki Ghostscript di bawahnya — server yang sama tetap rentan lewat jalur lain sampai Ghostscript-nya di-update.
- **Roundcube** bergantung pada kebijakan delegate ImageMagick yang mengizinkan handoff ke Ghostscript. Kalau delegate itu dimatikan, jalur ini tertutup — tapi itu mitigasi konfigurasi, bukan fix.
- **Google** sudah memitigasi jalur yang kena di layanannya pada 10 November 2026, menurut pwn.ai.

pwn.ai **belum mempublikasikan PoC exploit-nya** maupun studi kasus per aplikasi, dan menyebut ada temuan Ghostscript lain dari riset yang sama — termasuk yang mencapai Dropbox — yang masih dalam proses perbaikan dan belum dipublikasikan.

## **<span style='color:#ff5555'>Versi yang Terdampak dan yang Sudah Diperbaiki</span>**
***

| Branch / distro | Terdampak | Diperbaiki |
|--------|-----------|------------|
| Ghostscript upstream | Semua rilis sebelum 10.08.0 | **10.08.0** (8 September 2026) |
| Debian 13 (Trixie) | — | **10.05.1~dfsg-1+deb13u2** (DSA-6516-1) |
| Ubuntu | — | USN-8949-1 (21 Oktober 2026, tujuh rilis) |
| Arch, Alpine, Gentoo, openSUSE Tumbleweed | — | Sudah merilis build yang diperbaiki |
| Debian 12 (Bookworm) | **Perlu dicek — sumber bertentangan per 22 November 2026** | Lihat catatan di bawah |
| Amazon Linux | Masih rentan | Fix masih pending |
| RHEL | Masih rentan | Bug tracking masih open |

> ⚠ **Catatan Debian 12:** dua sumber yang dibaca untuk draft ini bertentangan pada 22 November 2026 — satu mencatat Debian 12 masih rentan (paket `10.0.0~dfsg-11+deb12u8`, belum ada fix), satu lagi mencatat Debian 12 sudah diperbaiki lewat DLA-4914-1 (`10.0.0~dfsg-11+deb12u9`). Jangan mengandalkan tabel ini untuk Debian 12: cek langsung di security tracker Debian untuk CVE-2026-125396 sebelum menyatakan statusnya.

Perhatian yang sama seperti di postingan PHP November: **update host saja tidak cukup**. Ghostscript sering dibundel di dalam container image atau aplikasi — update paket di host tidak mengganti salinan Ghostscript yang dipakai worker di dalam container. Cek salinan yang benar-benar dipakai tiap layanan, bukan hanya binary di host.

## **<span style='color:#ff5555'>Deteksi dan Mitigasi</span>**
***

### 1. Patch Segera — Remediasi Utama

- Update Ghostscript ke **10.08.0** atau build distro yang sudah berisi backport fix-nya, sesuai tabel di atas
- **Restart semua proses yang memakai Ghostscript setelah update** — dan rebuild container image yang membundel Ghostscript, lalu buat ulang container-nya. Proses yang sudah berjalan tetap memakai binary lama sampai di-restart
- Untuk WordPress: pastikan juga sudah di **7.0.4** (atau backport yang setara) untuk menutup CVE-2026-65640
- Verifikasi setelah restart: cek versi Ghostscript yang benar-benar dipakai layanan itu, bukan hanya binary di host

### 2. Mitigasi Sementara Bila Patch Tertunda

- **Matikan delegate PostScript/EPS/PDF ke Ghostscript di ImageMagick** (`policy.xml`) di layanan yang tidak butuh merender dokumen itu — ini menutup jalur WordPress/Roundcube/MediaWiki yang lewat ImageMagick
- **Jangan biarkan CUPS queue terekspos ke jaringan yang tidak tepercaya** — shared queue yang menerima print job anonim adalah jalur tanpa login yang paling sederhana di sini
- Batasi siapa yang bisa upload file dan format apa yang diterima, dan proses upload di worker yang terisolasi dengan hak seminimal mungkin — perintah yang lolos sandbox tetap jalan sebagai akun proses rendering, jadi akun itu seharusnya tidak punya apa-apa untuk dicuri

### 3. Cara Cek Cepat

```bash
# Versi Ghostscript yang terpasang di host ini
gs --version

# Paket Ghostscript di Debian/Ubuntu
dpkg -l | grep ghostscript

# Cari salinan Ghostscript lain yang mungkin dibundel aplikasi/container
find /opt /srv /var/www -name 'gs' -o -name 'libgs*' 2>/dev/null | head -20

# Apakah ImageMagick mengizinkan delegate ke Ghostscript (cek policy.xml)
grep -i -A2 -B2 'ghostscript\|delegate.*ps\|coder.*pdf' /etc/ImageMagick-*/policy.xml 2>/dev/null
```

Tanda paling jelas dari eksploitasi yang berhasil: **proses Ghostscript (`gs`/`gsc`) yang men-spawn shell, interpreter scripting, atau tool download/network** seperti `sh`, `bash`, `python`, `curl`, atau `nc`. Ghostscript yang sedang mengonversi upload, thumbnail, atau print job tidak punya alasan sah untuk membuat child process seperti itu — temuan seperti itu berarti sandbox kemungkinan sudah lolos. Jangan menjalankan PoC atau test exploit publik terhadap server produksi sebagai metode inventarisasi.

## **<span style='color:#ff5555'>Timeline</span>**
***

| Tanggal | Peristiwa |
|---------|-----------|
| 8 Sep 2026 | Artifex merilis Ghostscript 10.08.0, memperbaiki CVE-2026-125396 |
| 12 Agu 2026 | WordPress 7.0.4 menutup jalur upload-nya sendiri (CVE-2026-65640) — sebelum CVE Ghostscript-nya dipublikasikan |
| 21 Okt 2026 | Ubuntu merilis USN-8949-1 untuk tujuh rilis |
| 9 Nov 2026 | CVE-2026-125396 dipublikasikan (VulnCheck sebagai CNA) |
| 10 Nov 2026 | Google memitigasi jalur yang kena di layanan pemrosesan dokumennya |
| 17 Nov 2026 | pwn.ai (Paulos Yibelo) mempublikasikan riset Upload2Shell |
| 22 Nov 2026 | The Hacker News meliput riset ini; belum ada laporan eksploitasi di alam liar, belum masuk CISA KEV |

## **<span style='color:#ff5555'>Kesimpulan</span>**
***

Upload2Shell berbahaya bukan karena bug-nya eksotis — use-after-free di font loader — melainkan karena jaraknya: Ghostscript duduk di belakang fitur yang terasa tidak berbahaya, preview gambar, thumbnail email, dan print job. Penyerang tidak perlu mengekspos Ghostscript secara langsung; cukup mengirim file yang terlihat biasa ke fitur yang memang dirancang untuk menerima file dari orang luar.

Tiga hal yang perlu diingat: skor `AV:L` di CVSS jangan dibaca sebagai "harus akses lokal" — jalur-jalurnya bisa dijangkau dari jaringan; patch WordPress saja tidak memperbaiki Ghostscript di bawahnya, dan update host saja tidak memperbaiki salinan yang dibundel di container; dan Debian 12, RHEL, serta Amazon Linux masih perlu dicek satu per satu per 22 November — jangan berasumsi sudah aman karena upstream sudah merilis fix sejak September.

Periksa `gs --version` di semua tempat dokumen dan gambar diproses, update ke 10.08.0 atau build distro yang sudah ter-patch, dan lakukan sebelum celah ini berpindah dari riset menjadi eksploitasi di alam liar.

## **<span style='color:#ff5555'>Referensi</span>**
***

- The Hacker News, "Researchers Demonstrate Ghostscript Sandbox-Escape RCE via Crafted File Uploads in Popular Apps" — https://thehackernews.com/2026/11/researchers-demonstrate-ghostscript.html
- Cyber Security News, "Hackers Could Execute Commands on WordPress, Email and Print Servers Through Upload2Shell Flaw" — https://cybersecuritynews.com/upload2shell-flaw/
- Cyber Kendra, "Upload2Shell Ghostscript Bug Turns File Uploads Into Remote Code Execution, Patch Gaps Remain" — https://www.cyberkendra.com/2026/11/upload2shell-ghostscript-bug-turns-file-uploads-into-remote-code-execution-patch-gaps-remain.html
- CyberSixt, "Upload2Shell: Ghostscript CVE-2026-125396 lets uploads run code" — https://cybersixt.com/originals/upload2shell-ghostscript-cve-2026-125396
- CVE.org, CVE-2026-125396 — https://www.cve.org/CVERecord?id=CVE-2026-125396
- Artifex Bug 709684, fix commit 1484854e (dirujuk via liputan di atas)
