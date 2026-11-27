---
title: Idunno.AntiCsrf CVE-2026-131914 - Library Anti-CSRF yang Mati Tahun 2008, RCE Tanpa Auth dan Sedang Dieksploitasi
author: Thyrh07
date: 2026-11-24 08:00:00 +0700
categories: [Research]
tags: [Idunno.AntiCsrf, CVE-2026-131914, Mandiant, ASP.NET, WebForms, RCE, Deserialization, CWE-502]
excerpt: ""
---

>   **You are solely responsible for any actions or activities related to the material on this website. Some of the content here may be harmful or dangerous if misused. If you do not fully understand any material on this site, please leave. Before accessing or using these materials in any way, review the laws that apply in your province or country. All materials are provided for educational and research purposes only.**


## **<span style='color:#ff5555'>Ringkasan</span>**
***

Ini bukan cerita tentang library baru yang baru saja rusak — ini tentang library yang sudah mati selama 18 tahun dan ternyata masih jalan di server orang, lalu berubah menjadi pintu RCE tanpa autentikasi. `Idunno.AntiCsrf` adalah modul anti-CSRF pihak ketiga untuk ASP.NET WebForms yang rilis terakhirnya Desember 2008. Mandiant menemukannya saat investigasi incident response, dan kalimat yang paling menentukan ada di advisory mereka: *"Active exploitation in the wild has been observed."*

- **Nama:** Idunno.AntiCsrf — deserialisasi tak aman di modul anti-CSRF lawas ASP.NET WebForms
- **CVE:** CVE-2026-131914, CVSS 4.0 **9.3** (`CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N/E:A`), CWE-502 Deserialization of Untrusted Data
- **Penemu / pelapor:** LDB, Genwei Jiang, Dhanesh Kizhakkinan, Brian Bahtiarian, dan Lucas Kossack (Google Cloud / Mandiant Incident Response) — Mandiant MNDT-2026-0048, advisory dan CVE dipublikasikan 18 November 2026; diliput The Hacker News 23 November 2026
- **Versi terdampak:** 0.9.1.0 dan 0.9.1 Beta — seluruh rilis yang pernah ada, karena proyek ini tidak pernah merilis versi berikutnya
- **Target:** Aplikasi ASP.NET WebForms apa pun yang masih memuat `Idunno.AntiCsrf.dll` dan mendaftarkan `AntiCsrfModule` di `Web.config`
- **Fix:** **Tidak ada, dan tidak akan ada.** Proyek defunct, tidak ada maintainer, tidak ada patch resmi. Satu-satunya mitigasi adalah menghapus modulnya dan mengganti dengan anti-CSRF yang masih didukung
- **Status per 25 Nov 2026:** Eksploitasi aktif teramati di alam liar; pelaku dan korban tidak diungkap; **masuk CISA KEV 24 Nov 2026 (deadline federal 27 Nov 2026)**

> ⚠ **Klarifikasi penting:** Ini **bukan** kerentanan di framework ASP.NET atau IIS — yang rentan adalah library pihak ketiga `Idunno.AntiCsrf` yang ditempel ke aplikasi WebForms. Jangan juga membaca "library mati 2008" sebagai "tidak ada yang masih pakai" — justru di situlah bahayanya: DLL ini biasanya masuk ke folder `bin/` lewat copy-paste dari tutorial lama, tidak tercatat di NuGet (library ini memang tidak pernah dipublikasikan ke NuGet), jadi tidak muncul di SBOM dan tidak dikenal scanner kerentanan sampai CVE ini ada. Dan jangan mengandalkan hash file untuk mencari: Mandiant menegaskan build hasil kompilasi ulang dari source yang sama punya hash berbeda tapi tetap menyimpan sink yang rentan — cari berdasarkan **nama** (`Idunno.AntiCsrf` / `AntiCsrfModule`) di `Web.config` dan folder `bin/`, bukan berdasarkan hash.

## **<span style='color:#ff5555'>Cara Kerja Celahnya</span>**
***

`Idunno.AntiCsrf` menerapkan perlindungan CSRF model double-submit cookie: cookie `__CSRFCOOKIE` berisi GUID, dan form menyertakan field tersembunyi `__CSRFTOKEN` yang nilainya adalah hasil serialisasi GUID yang sama. Saat request kembali, modul membandingkan keduanya. Ide yang masuk akal — eksekusinya yang fatal.

`AntiCsrfModule` mendaftarkan diri ke event `PreRequestHandlerExecute` di pipeline `HttpApplication`. Event ini jalan **sebelum** `Page_Load` dan sebelum lifecycle halaman WebForms apa pun. Untuk setiap request `POST` ke endpoint `System.Web.UI.Page`, modul hanya mengecek satu hal: apakah cookie `__CSRFCOOKIE` dan field form `__CSRFTOKEN` keduanya tidak kosong. Tidak ada pengecekan apakah cookie itu GUID yang valid, tidak ada syarat halaman punya `<form runat="server">`.

Lalu, sebelum membandingkan apa pun, modul menjalankan `new ObjectStateFormatter().Deserialize()` terhadap isi field form — memakai `ObjectStateFormatter` tanpa parameter, tanpa MAC, tanpa signature, tanpa enkripsi, tanpa validasi `machineKey`, dan tanpa batasan `SerializationBinder`.

Artinya: penyerang cukup mengirim **satu HTTP POST tanpa autentikasi dan tanpa token CSRF yang valid** — cookie apa pun yang tidak kosong (misalnya `Cookie: __CSRFCOOKIE=1`) plus gadget chain `ObjectStateFormatter` yang di-encode Base64 di field `__CSRFTOKEN` — dan kode dieksekusi di dalam `Deserialize()`, sebagai proses worker IIS (`w3wp.exe`). Itu kompromi penuh server aplikasi web. Setelah itu modul melempar `PotentialCsrfException` (HTTP 500, atau redirect 302 kalau `detectionResult="Redirect"` atau `customErrors` aktif) — tapi sudah terlambat, kode penyerang sudah jalan duluan.

Yang membuat celah ini lebih luas dari yang terlihat: aturan autentikasi pipeline (`AuthenticateRequest` / `AuthorizeRequest`) memang jalan lebih dulu, tapi halaman `.aspx` apa pun yang bisa dijangkau anonim — atau halaman yang hanya mengandalkan pengecekan session di dalam `Page_Load` — tetap mengekspos sink ini, karena sink-nya dieksekusi sebelum `Page_Load` berjalan. Satu-satunya pengecualian adalah halaman yang mengimplementasikan `ISuppressCsrfCheck` atau didekorasi `[SuppressCsrfCheck]`.

## **<span style='color:#ff5555'>Kenapa Library 2008 Masih Jalan di 2026</span>**
***

Proyek AntiCSRF diterbitkan di CodePlex — platform hosting proyek open-source Microsoft yang sudah pensiun dan menjadi read-only sejak 2017. Rilis terakhir, 0.9.1 Beta, bertanggal 15 Desember 2008. Tidak ada penerus, tidak ada fork resmi, tidak ada maintainer.

Tapi DLL tidak mati hanya karena proyeknya mati. Di banyak organisasi, `Idunno.AntiCsrf.dll` masuk ke aplikasi lewat tutorial atau blog post tahun 2009, di-copy ke folder `bin/`, didaftarkan di `Web.config`, lalu dilupakan — sementara aplikasinya sendiri terus di-maintenance, di-upgrade framework-nya, dan dipindah-pindah server selama belasan tahun. Tidak ada yang memiliki dependensi ini, tidak ada yang memperbarui, dan tidak ada scanner yang mengenal namanya.

Mandiant sendiri menemukan celah ini bukan dari audit proaktif, melainkan saat investigasi incident response — analisis root cause selesai 4 November 2026, penugasan CVE dimulai 11 November, advisory dan CVE dipublikasikan 18 November. Itu detail yang penting: celah ini ditemukan karena ada insiden yang sedang ditangani, bukan karena seseorang iseng mengaudit library museum.

## **<span style='color:#ff5555'>Deteksi dan Mitigasi</span>**
***

Tidak ada patch — jangan menunggu. Yang ada adalah pekerjaan penghapusan:

**1. Cari dulu, berdasarkan nama — bukan hash.**

Audit semua file `Web.config` dan semua folder `bin/` di semua aplikasi ASP.NET untuk referensi `Idunno.AntiCsrf` atau `AntiCsrfModule`. Modul terdaftar sebagai `Idunno.AntiCsrf.AntiCsrfModule, Idunno.AntiCsrf` di `<httpModules>` / `<modules>`, dengan konfigurasi `<csrfSettings>` (`cookieName` default `__CSRFCOOKIE`, `formFieldName` default `__CSRFTOKEN`).

Hash berikut hanya berlaku untuk build yang diketahui — build hasil kompilasi ulang bisa berbeda:

| Build | Ukuran | MD5 | SHA-256 |
|---|---|---|---|
| `Idunno.AntiCsrf.dll` 0.9.1.0, rilis resmi 2008 | 69.632 byte | `dba68f9a091242b70b31f12caf90fdf3` | `da1efcfed182597ee946a9ef2895dfdfeab2bdff16855aeef01656d8158ead06` |
| Rebuild dari source 0.9.1 Beta, Release | 69.632 byte | `6160ca8df91abd964c93ea98216e81f9` | `c5c994891783453ae85f3ef8efb48592b7c560aeb56f531ee9ea5736518246eb` |
| Rebuild dari source 0.9.1 Beta, Debug | 70.144 byte | `48acb50c210ede14de69efaf035465ff` | `aa7bbad4a7ebc38519955ac65517716569508ee4f61fc2f1276086940501e9bf` |

**2. Hapus modulnya sampai bersih.**

Hapus semua entri `<add name="..." type="Idunno.AntiCsrf.AntiCsrfModule, Idunno.AntiCsrf" />` dari `<httpModules>` dan `<modules>`, hapus `<section name="csrfSettings" ... />` dari `<configSections>` dan elemen `<csrfSettings ... />` dari `Web.config`, hapus referensi kode ke `ISuppressCsrfCheck` / `[SuppressCsrfCheck]`, dan hapus `Idunno.AntiCsrf.dll` dari folder `bin/`.

**3. Ganti dengan anti-CSRF yang masih didukung — jangan biarkan form tanpa perlindungan.**

Mandiant merekomendasikan: `System.Web.Helpers.AntiForgery` (`AntiForgery.GetHtml()` dan `AntiForgery.Validate()`), `Page.ViewStateUserKey` dikombinasikan dengan ViewState yang ter-MAC / terenkripsi pada form postback, atau `IAntiforgery` di ASP.NET Core.

**4. Periksa apakah sudah dieksploitasi.**

Cek log IIS untuk request `POST` aneh ke endpoint `.aspx` yang mengembalikan HTTP 302 atau HTTP 500 dengan `cs-bytes` yang besar (bila field itu tercatat). Lingkungan yang punya bukti eksploitasi harus memutar semua secret yang tersimpan di `Web.config` dan `applicationHost.config` — termasuk kunci `<machineKey>`, kunci enkripsi, dan kredensial database — serta memeriksa webroot untuk file yang tidak sah.

## **<span style='color:#ff5555'>Timeline</span>**
***

- **15 Des 2008:** `Idunno.AntiCsrf` 0.9.1 Beta dirilis di CodePlex — rilis terakhir proyek ini
- **Nov 2017:** CodePlex menjadi read-only; proyek AntiCSRF tidak punya penerus
- **4 Nov 2026:** Analisis root cause selesai oleh FLARE selama investigasi incident response
- **11 Nov 2026:** Mandiant (sebagai CNA) memulai penugasan CVE — proyek upstream sudah defunct, tidak ada vendor atau maintainer yang bisa memberi patch
- **18 Nov 2026:** Advisory Mandiant MNDT-2026-0048 dan CVE-2026-131914 dipublikasikan; eksploitasi aktif di alam liar sudah teramati
- **23 Nov 2026:** The Hacker News meliput — Mandiant menilai exploitability "High", eksploitasi hanya butuh satu HTTP POST tanpa autentikasi
- **24 Nov 2026:** Masuk CISA KEV (deadline federal 27 Nov 2026) — konfirmasi resmi eksploitasi aktif; tetap belum ada patch; pelaku dan korban tidak diungkap

## **<span style='color:#ff5555'>Referensi</span>**
***

- Mandiant MNDT-2026-0048 — `Idunno.AntiCsrf` insecure deserialization (CVE-2026-131914): https://github.com/mandiant/Vulnerability-Disclosures/blob/master/2026/MNDT-2026-0048.md
- The Hacker News, 23 Nov 2026 — "Hackers Exploit Critical ASP.NET Anti-CSRF Library Flaw Allowing Unauthenticated RCE": https://thehackernews.com/2026/11/hackers-exploit-critical-aspnet-anti.html
- CVE.org — CVE-2026-131914: https://www.cve.org/CVERecord?id=CVE-2026-131914
- CodePlex Archive — AntiCSRF: https://codeplexarchive.org/project/AntiCSRF
