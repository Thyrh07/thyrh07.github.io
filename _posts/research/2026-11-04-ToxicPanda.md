---
title: ToxicPanda 2.0 - Banking Trojan yang Menyalahgunakan VPN, Accessibility, dan ADB untuk On-Device Fraud
author: Rhovelz
date: 2026-11-04 08:00:00 +0700
categories: [Research]
tags: [Android, Malware, BankingTrojan, RAT, MobileSecurity, OnDeviceFraud]
image: /assets/img/Post/ToxicPanda.png
excerpt: ""
---

>   **You are solely responsible for any actions or activities related to the material on this website. Some of the content here may be harmful or dangerous if misused. If you do not fully understand any material on this site, please leave. Before accessing or using these materials in any way, review the laws that apply in your province or country. All materials are provided for educational and research purposes only.**


## **<span style='color:#ff5555'>Ringkasan</span>**
***

ToxicPanda 2.0 adalah Android banking trojan generasi baru yang dipublikasikan oleh **Zimperium zLabs** pada **19 Agustus 2026**. Varian ini merupakan evolusi signifikan dari ToxicPanda asli (juga dikenal sebagai TgToxic) yang pertama kali didokumentasikan oleh Cleafy pada November 2024. ToxicPanda 2.0 dirancang untuk **on-device fraud (ODF)** — melakukan aksi penipuan langsung dari perangkat korban, bukan dari mesin attacker, sehingga lebih sulit dideteksi oleh sistem anti-fraud perbankan.

- **Target:** 349 aplikasi banking, financial, e-wallet, dan cryptocurrency di **16 negara**
- **Remote commands:** 167 perintah yang dapat dieksekusi dari jarak jauh
- **Teknik utama:** VPN abuse, Accessibility Service abuse, Wireless Debugging (ADB) self-pairing
- **Distribusi:** Amazon Web Services (AWS) S3 buckets, Telegram, situs palsu
- **Tujuan:** Account takeover dan on-device fraud

## **<span style='color:#ff5555'>Evolusi: Dari TgToxic ke ToxicPanda 2.0</span>**
***

ToxicPanda bukan malware baru. Keluarga ini telah berevolusi melalui beberapa tahap:

| Versi | Tahun | Peneliti | Target | Infeksi |
|-------|-------|----------|--------|---------|
| TgToxic | 2022 | Trend Micro | Asia Tenggara | Awal kemunculan |
| ToxicPanda 1.0 | 2024 | Cleafy | 16 banking apps, Eropa & LATAM | 1.500+ perangkat |
| ToxicPanda 2.0 | 2026 | Zimperium zLabs | 349 apps, 16 negara | Belum dipublikasikan |

Kampanye ToxicPanda 1.0 yang didokumentasikan Cleafy menunjukkan lebih dari **1.500 perangkat terinfeksi**, dengan distribusi infeksi:

- **Italia** — lebih dari separuh infeksi
- **Portugal** — target utama kedua
- **Spanyol, Prancis, Peru** — target tambahan

ToxicPanda 2.0 memperluas jangkauan secara drastis: dari 16 banking apps menjadi **349 target** di 16 negara, dan dari segelintir perintah menjadi **167 remote commands**.

## **<span style='color:#ff5555'>Arsitektur: Dropper + Payload</span>**
***

ToxicPanda 2.0 beroperasi sebagai **dropper** yang mengekstrak dan menginstall payload terenkripsi. Kunci dari desainnya adalah urutan serangan yang memanipulasi fitur-fitur Android yang sah — VPN, Accessibility, Developer Options, dan ADB — dalam rangkaian yang membuat setiap langkah terlihat legitimate secara individual.

### Alur Infeksi

```
[Distribusi]
     │
     ├── APK di-host di AWS S3 buckets / Telegram / situs palsu
     │
     └── Korban menginstall dan menjalankan dropper
          │
          ├── Tampilkan fake installation screen (WebView)
          ├── Minta izin VPN
          │   └── VPN lokal memblokir traffic ke Google Play & Play Services
          │
          ├── Dekripsi dan install encrypted payload dari assets
          ├── Minta izin Accessibility Service
          │
          ├── [Accessibility] Buka Settings → Device Info
          │   └── Tap "Build Number" 7 kali → Aktifkan Developer Options
          │
          ├── [Accessibility] Navigasi ke Wireless Debugging
          │   ├── Aktifkan fitur
          │   ├── Buka "Pair device with pairing code"
          │   └── Baca 6-digit pairing code + dynamic port dari UI
          │
          ├── Pair dengan ADB daemon lokal (127.0.0.1)
          │   └── Gunakan SPAKE2/TLS untuk autentikasi
          │
          └── Dapatkan ADB shell user privileges
              ├── Grant runtime permissions
              ├── Relax background limits
              ├── Enable komponen malicious
              └── Establish persistence
```

## **<span style='color:#ff5555'>Teknis: VPN Abuse untuk Memblokir Google Play</span>**
***

Langkah pertama yang dilakukan ToxicPanda 2.0 setelah instalasi adalah meminta **izin VPN**. Permintaan ini terlihat wajar — banyak aplikasi legitimate yang menggunakan VPN. Namun di Android, VPN lokal memiliki kemampuan untuk **mengontrol traffic mana yang diizinkan keluar dari perangkat**.

ToxicPanda memanfaatkan posisi ini untuk:

1. **Memblokir komunikasi ke Google Play** — mencegah Play Store melakukan verifikasi atau update
2. **Memblokir Google Play Services** — mencegah Play Protect dari mendeteksi payload malicious
3. **Mengisolasi perangkat** dari mekanisme keamanan Google selama fase instalasi payload

> ⚠ **Penting:** Zimperium tidak mengklaim bahwa langkah VPN ini secara permanen menonaktifkan Google Play Protect. Fakta yang terverifikasi adalah dropper memblokir komunikasi ke Google Play dan Google Play Services **selama fase instalasi payload**, mengurangi kesempatan platform untuk mendeteksi atau merespons.

## **<span style='color:#ff5555'>Teknis: ADB Self-Pairing via Accessibility</span>**
***

Ini adalah inovasi paling menonjol dari ToxicPanda 2.0. Malware ini menjadi **kedua sisi** dari proses pairing ADB — mengontrol antarmuka Android melalui Accessibility sambil bertindak sebagai "development workstation" yang mengonsumsi pairing secret.

### Proses Detail

| Langkah | Aksi | Mekanisme |
|---------|------|-----------|
| 1 | Buka Device Information | Accessibility Service |
| 2 | Tap "Build Number" 7 kali | Accessibility Service (mengaktifkan Developer Options) |
| 3 | Navigasi ke Wireless Debugging | Accessibility Service |
| 4 | Aktifkan Wireless Debugging | Accessibility Service |
| 5 | Buka dialog "Pair device with pairing code" | Accessibility Service |
| 6 | Baca 6-digit code + dynamic port | Accessibility (membaca UI hierarchy) |
| 7 | Generate key pair | Aplikasi malware |
| 8 | Pair dengan ADB daemon di 127.0.0.1 | SPAKE2/TLS authentication |
| 9 | Koneksi sebagai `shell` user | ADB protocol |

Setelah mendapatkan **ADB `shell` user privileges**, ToxicPanda menjalankan perintah untuk:

- `pm grant` — memberikan runtime permissions tanpa interaksi pengguna
- Merelaksasi background execution limits
- Mengaktifkan komponen malicious yang sebelumnya disabled
- Membangun persistence

> **Catatan:** Privilege yang didapat adalah ADB `shell` user — **bukan root access**. Ini tetap memberikan kontrol yang signifikan karena banyak operasi sensitif dapat dilakukan dengan shell privileges.

## **<span style='color:#ff5555'>Kapabilitas: Pencurian Kredensial & Remote Control</span>**
***

### Phishing Overlay

ToxicPanda 2.0 menggunakan sistem overlay yang dinamis dan dikontrol dari server:

1. Malware mengirimkan **daftar package names dan ikon** aplikasi terinstall ke C2 server
2. C2 mengidentifikasi aplikasi target di antara 349 yang didukung
3. Ketika korban membuka aplikasi target, C2 mengirimkan **fake HTML screen**
4. Malware menampilkan overlay di atas layar login atau transaksi yang sebenarnya
5. **Transparent overlay** menangkap posisi sentuhan untuk mencuri PIN aplikasi finansial

### PIN & Lock Screen Theft

Selain overlay banking, ToxicPanda memiliki modul terpisah:

- **PIN Harvesting Module** — memonitor lebih dari **140 aplikasi finansial dan cryptocurrency**
- **Fake Lock Screen** — meniru tampilan lock screen Android untuk mencuri PIN, pola, dan password perangkat
- **Fake System Update** — menampilkan layar update palsu sementara aktivitas malicious berjalan di background

### 167 Remote Commands

ToxicPanda 2.0 mengimplementasikan 167 remote commands. Beberapa kategori utama:

| Kategori | Contoh Command | Deskripsi |
|----------|---------------|-----------|
| **ADB Control** | `adbShell` | Eksekusi shell command arbitrary melalui ADB |
| **ADB Control** | `adbClick`, `adbSwipe` | Simulasi tap dan swipe gesture |
| **ADB Control** | `adbScreenshot` | Capture, compress, dan Base64-encode screenshot |
| **Screen Manipulation** | `adbBlackScreen` | Set brightness ke 0 untuk menyembunyikan aktivitas |
| **Screen Manipulation** | `adbLockAndroidUpdate` | Overlay fake update screen menggunakan shell user |
| **Lock Control** | `admLock` | Matikan layar |
| **Lock Control** | `admPwd` | Force reset password/PIN perangkat |
| **Device Control** | `dnd` | Aktifkan Do Not Disturb untuk menyembunyikan notifikasi |
| **Device Control** | `disablePocketMode` | Nonaktifkan proteksi sentuhan tidak sengaja |
| **Persistence** | `adm`, `admSet` | Request Device Administrator rights |
| **Persistence** | `antiDeleteOff` | Kontrol mekanisme anti-penghapusan |
| **Data Theft** | `addPinTargets` | Update daftar target secara dinamis dari server |

> **Catatan:** 167 commands diimplementasikan dalam kode, tetapi ini tidak berarti semua digunakan dalam serangan nyata. Laporan publik tidak mengonfirmasi eksekusi setiap command di lingkungan production.

## **<span style='color:#ff5555'>Komunikasi C2</span>**
***

ToxicPanda 2.0 menggunakan skema komunikasi dua tahap:

```
[Perangkat Terinfeksi]
     │
     ├── 1. HTTPS Request awal → C2 Server
     │      └── Registrasi perangkat, kirim info device
     │
     └── 2. Persistent WebSocket Connection
            ├── Terenkripsi dengan AES-ECB
            ├── Key di-generate dari static byte arrays dalam kode
            ├── Menerima commands dari C2
            └── Mengirim data curian ke C2
```

### Kelemahan Kriptografi

Penggunaan **AES-ECB** (Electronic Codebook) mode merupakan kelemahan yang notable:

- ECB mengenkripsi setiap blok secara independen — pola dalam plaintext terlihat di ciphertext
- Key di-generate dari **static byte arrays** yang hard-coded dalam kode — memungkinkan ekstraksi melalui reverse engineering
- Ini membuat traffic C2 dapat di-decrypt oleh defender yang memiliki sample malware

## **<span style='color:#ff5555'>Distribusi</span>**
***

Berbeda dengan malware yang menyusup ke Google Play Store, ToxicPanda 2.0 didistribusikan melalui **sideloading**:

| Metode | Detail |
|--------|--------|
| **AWS S3 Buckets** | Payload di-host di beberapa region: `ap-south-1`, `mx-central-1`, `ap-southeast-1` |
| **Telegram** | APK didistribusikan melalui Telegram bot file sharing |
| **Situs Palsu** | Domain seperti `googleplaydown[.]com` meniru halaman download Play Store |

Metode spesifik untuk menipu pengguna agar mengunduh file awal belum dipublikasikan secara detail.

## **<span style='color:#ff5555'>Deteksi</span>**
***

### Sinyal Deteksi

Kunci deteksi ToxicPanda 2.0 adalah **mengenali rangkaian aksi**, bukan hanya satu indikator tunggal:

1. **Sekuens Instalasi** — VPN registration → hilangnya konektivitas Google services → Accessibility activation → Developer Options changes → Wireless ADB pairing → battery optimization exceptions
2. **VPN Profile** — Aplikasi yang tidak dikenal membuat profil VPN dan memblokir traffic Google Play
3. **Accessibility Abuse** — Service accessibility diberikan ke aplikasi yang tidak diotorisasi
4. **ADB Pairing** — Wireless Debugging diaktifkan dan pairing baru dibuat tanpa inisiasi pengguna
5. **Network Pattern** — Traffic shifting dari HTTPS ke persistent WebSocket connection

### Observable Logs

```
# Endpoint / EDR
- Download dan instalasi APK yang tidak diotorisasi
- Dekripsi dan unpacking payload dari assets
- VPN Service, Accessibility Service, Device Administrator requests dari app tidak dikenal
- Aktivasi Developer Options dan Wireless Debugging diikuti local ADB pairing
- ADB shell commands: pm grant, perubahan background limits, component activation
- Fake login screens (WebView), transparent overlays, fake lock screens

# Proxy / SWG / DNS
- Akses ke AWS distribution URLs yang tercantum dalam IOC
- Traffic shifting dari HTTPS ke persistent WebSocket connection
- Long-running WebSockets dari Android devices yang tidak sesuai penggunaan normal

# MDM / Mobile Threat Defense
- Korelasi perubahan cepat pada VPN, Accessibility, Developer Options, dan Wireless Debugging
```

### Hunting Queries

```bash
# Cek Accessibility services aktif (via ADB)
adb shell settings get secure enabled_accessibility_services

# Cek Developer Options status
adb shell settings get global development_settings_enabled

# Cek Wireless Debugging status
adb shell settings get global adb_wifi_enabled

# List VPN profiles aktif
adb shell dumpsys connectivity | grep -A5 "VPN"

# Cek Device Administrator aktif
adb shell dumpsys device_policy | grep -A3 "admin"

# Monitor ADB connections
adb shell cat /data/misc/adb/adb_keys 2>/dev/null
```

> ⚠ **Penting:** Jangan mengonfirmasi infeksi hanya berdasarkan IOC match. Verifikasi secara bertahap: payload unpacking, permission grants, komunikasi C2, dan bukti aktivitas ADB.

## **<span style='color:#ff5555'>Mitigasi</span>**
***

### Untuk Pengguna

1. **Jangan install APK** dari sumber di luar Google Play Store
2. **Tolak permintaan VPN** dari aplikasi yang baru diinstall — terutama selama proses instalasi
3. **Tolak permintaan Accessibility Service** dari aplikasi yang tidak dikenal
4. **Jangan aktifkan Developer Options** kecuali benar-benar diperlukan untuk pengembangan
5. Aktifkan dan perbarui **Google Play Protect** secara berkala
6. Periksa dan cabut **Device Administrator** dari aplikasi yang tidak dikenali

### Untuk Administrator / SOC

1. **Gunakan MDM** untuk merestriksi sideloading, Accessibility Service yang tidak diotorisasi, VPN profiles yang tidak diotorisasi, dan Developer Options / Wireless Debugging
2. **Deploy Mobile Threat Defense (MTD)** untuk korelasi APK, permissions, overlays, ADB, dan C2 pada perangkat
3. **Monitor rangkaian aksi** — VPN registration + Accessibility + Developer Options + ADB pairing sebagai satu alur serangan
4. Blokir akses ke **IOC network indicators** di DNS dan proxy level
5. **Edukasi pengguna** untuk tidak menyetujui permintaan VPN atau Accessibility selama instalasi aplikasi

### Untuk Banking App Developer

1. Implementasikan **server-side fraud detection** yang tidak bergantung pada integritas perangkat
2. Gunakan **transaction verification** dan **device binding** independen
3. Gunakan **phishing-resistant authentication** (FIDO2/WebAuthn)
4. **Jangan bergantung hanya pada keputusan client-side** — ToxicPanda dapat mengontrol seluruh UI perangkat

## **<span style='color:#ff5555'>MITRE ATT&CK Mapping (Mobile)</span>**
***

| Technique ID | Nama | Confidence |
|-------------|------|------------|
| T1453 | Abuse Accessibility Features | High |
| T1516 | Input Injection | High |
| T1059.004 | Command and Scripting Interpreter: Unix Shell | High |
| T1417.002 | Input Capture: GUI Input Capture | High |
| T1437.001 | Application Layer Protocol: Web Protocols | High |
| T1521.001 | Encrypted Channel: Symmetric Cryptography | High |
| T1406 | Obfuscated Files or Information | High |
| T1418 | Software Discovery | High |
| T1629.003 | Impair Defenses: Disable or Modify Tools | Medium |
| T1626.001 | Abuse Elevation Control Mechanism: Device Administrator Permissions | Medium |
| T1646 | Exfiltration Over C2 Channel | High |

## **<span style='color:#ff5555'>Timeline</span>**
***

| Tanggal | Event |
|---------|-------|
| Juli 2022 | TgToxic pertama kali teridentifikasi di Asia Tenggara |
| November 2024 | Cleafy mempublikasikan analisis ToxicPanda 1.0 — 1.500+ infeksi di Eropa & LATAM |
| 19 Agustus 2026 | Zimperium zLabs mempublikasikan laporan ToxicPanda 2.0 |
| 23 Agustus 2026 | BleepingComputer dan media keamanan lainnya mempublikasikan liputan |

## **<span style='color:#ff5555'>Indicator of Compromise (IOC)</span>**
***

### C2 Infrastructure

```
hxxps://cctvv2[.]com/panel
hxxps://www[.]v17001[.]com/panel
hxxps://www[.]w17907[.]com/panel
hxxps://g8688808[.]com
```

### Distribution URLs

```
hxxp://xcmtxc1oam1.s3.ap-south-1.amazonaws[.]com
hxxps://ui920fht4m1.s3.mx-central-1.amazonaws[.]com/
hxxps://googleplaydown[.]com/dUyFpz0Ai7Sp
hxxps://api.telegram[.]org/file/bot8331125434:AAEsxWq7_xZWUjN0IFLO3fTU54ZCBnZS4YI/documents/file_22526.apk
hxxps://api.telegram[.]org/file/bot8978201863:AAG4vnsJVOAbABfF6dXhhM8YJ14pd5LaxVw/documents/file_1389.apk
hxxps://osy4u8pmu.s3.ap-southeast-1.amazonaws[.]com/
hxxps://ejjsi5fbl.s3.ap-southeast-1.amazonaws[.]com/
hxxps://svhc5arco.s3.ap-southeast-1.amazonaws[.]com/
```

### Sample Hashes — Dropper (SHA-256)

Zimperium merilis **59 dropper hashes**. Berikut sebagian:

```
d0b44b17f97097480fabd89c86a4dd3ed2e6e60c61a498f17bb722e3fb9cc0d7
82156990c18aa0b85970b9fa3c11e16545c4fc435ecebb92ed80c325dc83e953
ef48459e8c814944eb4d5b56339421084facfb6d36aef015da016b996a0d8e54
d82e6ff06aa1077ad2492aa1a9bd6e787d8a094521589039f1d3773e163ca59a
90e105bb10cabaef70e62680aae061ab731de649ff0503f42d52c78640a51215
1199c3cb7707938f48c2611ae29699ad80a16132654f287c957edb36ccd74553
e0f3bda1d5236c7881fe4447b1531d3c8c5456ecfef942640b9b0c1c3e470c1e
290b4f9a3297f52af2e9344bac0a3f2e13efdaa7d01c24b5bb6d47420ce9f73f
1aae4a54a4c1b136cd78342a2c0265eb926f066760219bfbf3ce5e4b6a2657bd
e59d1f0a0c708fd678a55e431fb9dd8f068887a459a7caf92ff310687821ffd1
```

### Sample Hashes — Payload APK (SHA-256)

Zimperium merilis **102 payload hashes**. Berikut sebagian:

```
3c76887d942a2a1c02455070887e14015ea9fe63fc9e4777bfbc633ba29601d5
6174d8867e02177913dd28e17e67aac03a0be80f6cf2640a53a2ec8dd68c8f03
6ca69f5ecdfefb9e267745407b4400534dcacf25b500ac88a40ad267ab86b336
5890f6378baf86d74d0cfdc9ed6a46f9e769da12512d645eb6f1d8a868b349ea
e8a4d5ad6fdf97ebd647f6e4ab218c18bb624114d445785b9579a80427ae6d67
da240b9352248448e3875c9a47679070c42801a0c426d2daba3854470d99bd43
6a622b79aa2634b0ce77b6d2126ab575a8225f79b4c4c3b05cf520738a2b874b
a6a9babbd8c3ffd0801ccdaded12b55b7a3195289337a22145f0e4ece7984cbf
35e7357982afa73e75e23b5be7e0cf550059f0a2b30cc499f60e24a8185abf6d
ce65703cd7cca1f110494588b697dbd1d682be4f162437b1ee03c1aad4b24692
```

> Daftar lengkap IOC tersedia di [Zimperium IOC Repository](https://github.com/Zimperium/IOC/tree/master/2026-08-ToxicPanda).

## **<span style='color:#ff5555'>Referensi</span>**
***

- Zimperium zLabs — [The ToxicPanda Never Sleeps: ToxicPanda 2.0 Prepares its Next Strike on Mobile](https://zimperium.com/blog/the-toxicpanda-never-sleeps-toxicpanda-2.0-prepares-its-next-strike-on-mobile)
- Zimperium — [ToxicPanda 2.0 IOC Repository](https://github.com/Zimperium/IOC/tree/master/2026-08-ToxicPanda)
- Cleafy Labs — [ToxicPanda: A New Banking Trojan from Asia Hit Europe and LATAM](https://www.cleafy.com/cleafy-labs/toxicpanda-a-new-banking-trojan-from-asia-hit-europe-and-latam)
- BleepingComputer — [ToxicPanda Android Malware Uses VPN Permissions to Block Google Play](https://www.bleepingcomputer.com/news/security/toxicpanda-android-malware-uses-vpn-permissions-to-block-google-play/)
- Malwarebytes — [ToxicPanda 2.0 Can Take Over Your Android Phone and Banking Apps](https://www.malwarebytes.com/blog/mobile/2026/08/toxicpanda-2-0-can-take-over-your-android-phone-and-banking-apps)
- Cyber Security News — [ToxicPanda Android Malware Can Steal Banking PINs and Gain Shell Access to Phones](https://cybersecuritynews.com/toxicpanda-android-malware/)
- Security Affairs — [ToxicPanda 2.0 Gets a Major Upgrade, Expanding Attacks Across 16 Countries](https://securityaffairs.com/197681/breaking-news/toxicpanda-2-0-gets-a-major-upgrade.html)
- SecurityWeek — [Banking Trojans Manic, Grandoreiro, ToxicPanda 2.0 in the Spotlight](https://www.securityweek.com/banking-trojans-manic-grandoreiro-toxicpanda-2.0-in-the-spotlight/)
