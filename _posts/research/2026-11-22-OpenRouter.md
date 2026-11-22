---
title: OpenRouter - Pencurian Prompt di 3 Provider AI, 2 Juta Canary Token Membongkar Rantai Pasok AI
author: Thyrh07
date: 2026-11-22 08:00:00 +0700
categories: [Research]
tags: [OpenRouter, Dragos, AI, Prompt-Theft, ARTEX, Zero-Data-Retention]
excerpt: ""
---

>   **You are solely responsible for any actions or activities related to the material on this website. Some of the content here may be harmful or dangerous if misused. If you do not fully understand any material on this site, please leave. Before accessing or using these materials in any way, review the laws that apply in your province or country. All materials are provided for educational and research purposes only.**


## **<span style='color:#ff5555'>Ringkasan</span>**
***

Riset canary token Dragos membuktikan prompt pelanggan dicuri di dalam tiga provider hosting inference AI selama berbulan-bulan — tanpa satu pun provider itu sadar sampai peneliti memberi tahu mereka. OpenRouter, gateway yang merutekan request pelanggan ke puluhan provider pihak ketiga, menindaklanjuti pada 22 November 2026: tiga provider dihapus dari platform, 125 endpoint dicabut status Zero Data Retention-nya, dan semua pelanggan yang mungkin terdampak diberitahu.

- **Nama:** Pencurian prompt OpenRouter / Dragos — tanpa CVE, ini bukan kerentanan software, melainkan kompromi provider dan pelanggaran kebijakan data
- **Penemu:** Dragos, dipimpin HD Moore (Chief Research Officer) — riset "Three breaches, endless leaks: Your AI supply chain needs canaries", dipublikasikan 19 November 2026
- **Metode riset:** ~2 juta prompt berisi canary token (link unik + kredensial AWS palsu) dikirim Juli–November 2026 ke layanan AI, lalu peneliti mengawasi di mana secret itu muncul dipakai
- **Terdampak:** 3 provider hosting inference dibobol penyerang; sejumlah marketplace seller lain sengaja me-log request pelanggan; jumlah pengguna atau percakapan yang terekspos tidak diungkap
- **Status per 22 Nov 2026:** OpenRouter sudah bertindak hari ini — 3 provider dihapus, 125 endpoint dicabut dari Zero Data Retention, pelanggan yang mungkin kena sudah dinotifikasi. Nama 3 provider yang dihapus TIDAK diungkap

> ⚠ **Klarifikasi penting:** OpenRouter sendiri TIDAK dibobol — yang dibobol adalah provider pihak ketiga di balik gateway-nya. Dan 3 provider yang sudah mempublikasikan laporan (Phala, Together AI, Novita) belum tentu sama dengan 3 provider yang dihapus OpenRouter — OpenRouter maupun Dragos tidak mengonfirmasi apakah keduanya kelompok yang sama. Jangan menyamakan keduanya.

## **<span style='color:#ff5555'>Cara Risetnya Bekerja</span>**
***

Dragos tidak menebak-nebak — mereka menanam umpan lalu menunggu umpan itu dipakai. Antara Juli dan November 2026, tim Dragos mengirim sekitar 2 juta prompt ke layanan AI yang banyak dipakai, masing-masing berisi canary token: URL unik dan kredensial AWS palsu yang hanya ada di satu request ke satu provider. Kalau kredensial itu kemudian dipakai untuk panggilan API, atau link unik itu diakses, peneliti tahu persis prompt mana yang bocor, lewat provider mana, dan kapan.

Hasilnya lebih buruk dari dugaan. Tiga provider hosting inference terbukti dibobol penyerang yang secara aktif mencuri data request pelanggan selama berbulan-bulan. Peneliti juga menemukan temuan kedua yang tidak butuh pembobolan sama sekali: seller di marketplace inference yang memasang harga termurah ternyata sengaja mengaktifkan logging request — mengumpulkan apa pun yang mengalir lewat mereka.

Yang paling mengganggu: tidak satu pun dari tiga provider yang dibobol itu tahu mereka sudah dimasuki sampai Dragos memberi tahu. Canary token yang ditanam masih diakses berbulan-bulan setelah pencurian pertama, sampai minggu riset ini dipublikasikan.

## **<span style='color:#ff5555'>Tiga Temuan Utama</span>**
***

### 1. Tiga provider dibobol, request pelanggan disalin berbulan-bulan

Ini temuan paling serius menurut Dragos. Penyerang menguasai infrastruktur di tiga provider dan menyalin request pelanggan — termasuk prompt yang berisi secret, kredensial, dan data sensitif lain. Insiden paling awal dimulai pertengahan Juli 2026.

Kasus terbesarnya: beberapa ratus kunci AWS yang ditanam Dragos, masing-masing dikirim dalam tepat satu request ke satu provider selama jendela singkat akhir Oktober 2026 — dan semuanya disalin. Semua request ke satu provider dalam jendela itu disalin, termasuk request yang ditolak model sebelum diproses. Kunci AWS curian itu lalu dipakai untuk puluhan ribu panggilan API, mencari secret dan storage lain.

### 2. ARTEX dipakai penyerang untuk mengecek kunci curian

Pada 1 November 2026, penyerang yang sama memakai ARTEX — framework penetration testing berbasis AI — untuk satu hal saja: mengecek apakah kunci yang sudah dicuri masih hidup. ARTEX di sini bukan alat pencurian; ia hanya dipakai sebagai pengecek validitas kredensial, seperti seorang penyerang mengecek kunci hasil curian masih bisa dipakai atau tidak.

Ini yang membuat kasus ini nyambung dengan riset sebelumnya di blog ini: ARTEX adalah framework yang sama yang pernah dibahas sebagai sampel tulisan — dan di insiden nyata ini, ia muncul sebagai perkakas pasca-pencurian, bukan sebagai penyebab kebocoran.

### 3. Zero Data Retention tidak selalu ditepati

Sejumlah request yang ditandai Zero Data Retention — kebijakan yang menjanjikan prompt dan jawaban tidak disimpan — ternyata tetap disimpan atau diteruskan. Satu lab AI Tiongkok yang tidak disebutkan namanya mengambil URL rahasia dari lebih separuh prompt yang dikirim ke modelnya antara Juli dan Oktober 2026, termasuk sekitar dua pertiga prompt yang ditandai no-data-collection atau Zero Data Retention.

OpenRouter merespons dengan mencabut 125 endpoint dari kelayakan Zero Data Retention — mencakup endpoint yang ditandai peneliti plus yang dipotong sebagai tindakan pencegahan. Artinya, angka 125 itu bukan 125 pelanggaran terkonfirmasi; sebagian adalah pemotongan preventif.

## **<span style='color:#ff5555'>Apa Kata Provider yang Sudah Melapor</span>**
***

Tiga provider sudah mempublikasikan laporan — tapi sekali lagi, belum tentu ketiganya adalah yang dihapus OpenRouter:

| Provider | Yang terjadi |
|----------|--------|
| Phala | Penyerang menguasai satu server inference dan bisa melihat data request saat sedang diproses. Insiden dikandung 8 November 2026. Request ke model DeepSeek V4 Flash 0731 yang sampai ke Phala lewat gateway antara 21 Oktober 00:00 UTC dan 8 November 05:35 UTC harus dianggap terekspos — semua key, token, password, atau secret di dalamnya harus di-rotate |
| Together AI | Kredensial bocor lewat coding agent yang commit ke repository GitHub publik, lalu pihak tak dikenal mengunduh sampel prompt pelanggan Juni–Juli 2026. Pemberitahuan keamanan diterbitkan 21 November 2026 oleh CEO Vipul Ved Prakash, menyebut lima kredensial canary Dragos yang diuji penyerang |
| Novita | Masalah konfigurasi routing mengirim sebagian request MiniMax ke API resmi MiniMax, di luar yang dimaksudkan untuk pelanggan tersebut. Dikonfirmasi lewat privacy notice minggu ini |

Tidak semua provider yang terdampak sudah melapor secara publik — OpenRouter menyatakan sudah memberitahu pengguna yang terdampak dengan cara apa pun.

## **<span style='color:#ff5555'>Kenapa Ini Penting</span>**
***

Kasus ini mengubah cara berpikir soal privasi AI. Selama ini, perhatian tertuju pada modelnya — apakah model menyimpan chat, apakah data dipakai untuk training. Riset Dragos menunjukkan rantai pasok di belakang model itu sama berbahayanya: prompt kamu melewati gateway, lalu provider hosting, lalu mungkin forwarder dan marketplace seller, sebelum sampai ke model. Setiap hop adalah tempat prompt bisa disalin, di-log, atau dicuri — dan label "Zero Data Retention" di satu lapisan tidak menjamin lapisan di bawahnya menepatinya.

Bagi siapa pun yang pernah mengirim secret lewat prompt — API key, token, password, kredensial cloud — ke provider pihak ketiga, terutama lewat OpenRouter, risikonya nyata dan sudah terbukti terjadi: kunci AWS palsu Dragos dipakai puluhan ribu kali setelah dicuri. Kunci asli akan dipakai dengan cara yang sama.

## **<span style='color:#ff5555'>Deteksi dan Mitigasi</span>**
***

- **Rotate semua secret/API key yang pernah dikirim lewat prompt** ke provider pihak ketiga, terutama via OpenRouter — anggap semua yang pernah ada di prompt sudah terekspos, bukan hanya yang dikirim ke provider yang disebutkan di atas
- **Cek notifikasi dari OpenRouter** — OpenRouter menyatakan sudah memberitahu semua pelanggan yang mungkin terdampak; kalau kamu menerima email soal MiniMax atau provider lain, ikuti instruksi rotate di dalamnya
- **Khusus pengguna Phala via OpenRouter:** request ke model DeepSeek V4 Flash 0731 antara 21 Oktober 00:00 UTC dan 8 November 05:35 UTC harus dianggap terekspos — rotate semua secret di request itu
- **Jangan kirim secret di prompt** — tidak ada API key, token, password, atau kredensial cloud di dalam prompt, ke provider mana pun, dengan label apa pun
- **Periksa log dan pemakaian kredensial cloud kamu** — kunci curian dipakai untuk panggilan API massal mencari secret/storage; panggilan AWS yang tidak dikenal dari periode Juli–November 2026 patut dicurigai
- **Tinjau kembali pilihan provider inference** — harga termurah di marketplace bisa berarti provider yang me-log request; Zero Data Retention di gateway tidak sama dengan Zero Data Retention di seluruh rantai

## **<span style='color:#ff5555'>Timeline</span>**
***

| Tanggal | Peristiwa |
|---------|-----------|
| Jul–Nov 2026 | Dragos mengirim ~2 juta prompt berisi canary token ke layanan AI |
| Pertengahan Jul 2026 | Insiden paling awal dari tiga pembobolan provider dimulai |
| Jun–Jul 2026 | Penyerang mengunduh sampel prompt Together AI setelah kredensial bocor via coding agent |
| Akhir Okt 2026 | Semua request ke satu provider dalam jendela singkat disalin, termasuk yang ditolak model; ratusan kunci AWS canary dicuri |
| 1 Nov 2026 | Penyerang memakai ARTEX untuk mengecek kunci curian masih hidup atau tidak |
| 8 Nov 2026 | Insiden Phala dikandung |
| 19 Nov 2026 | Dragos mempublikasikan riset "Three breaches, endless leaks" |
| 21 Nov 2026 | Together AI menerbitkan security notice; OpenRouter mulai mengirim email ke pelanggan soal rute MiniMax |
| 22 Nov 2026 | OpenRouter menghapus 3 provider dari platform, mencabut 125 endpoint dari Zero Data Retention, dan memberitahu pelanggan yang mungkin terdampak; The Hacker News dan media keamanan lain meliput serentak |

## **<span style='color:#ff5555'>Kesimpulan</span>**
***

Pencurian prompt ini bukan soal satu model yang bocor — ini soal rantai pasok AI yang tidak terlihat oleh pengguna. Prompt kamu tidak berhenti di gateway; ia melewati provider hosting, forwarder, dan marketplace yang masing-masing bisa menyimpan, meneruskan, atau kehilangan datanya. Dragos membuktikannya dengan cara yang tidak bisa dibantah: umpan yang hanya ada di satu request muncul dipakai di tempat lain, berbulan-bulan kemudian.

Tiga hal yang perlu diingat: OpenRouter sendiri tidak dibobol, tapi provider di belakangnya iya — dan nama provider yang dihapus tidak diungkap; label Zero Data Retention tidak menjamin seluruh rantai menepatinya; dan ARTEX yang muncul di insiden ini adalah perkakas pengecek kredensial pasca-pencurian, bukan penyebab kebocoran. Kalau kamu pernah mengirim secret lewat prompt, rotate sekarang — jangan tunggu notifikasi.

## **<span style='color:#ff5555'>Referensi</span>**
***

- Dragos (HD Moore), "Three breaches, endless leaks: Your AI supply chain needs canaries" (19 Nov 2026) — riset canary token, ~2 juta prompt Juli–November 2026
- OpenRouter, "Data Processing Incidents on Inference Providers" (22 Nov 2026) — 3 provider dihapus, 125 endpoint dicabut dari Zero Data Retention, pelanggan dinotifikasi
- The Hacker News, "OpenRouter Drops 3 Providers After Researchers Catch Months of Prompt Theft at AI Hosting Firms" (22 Nov 2026) — https://thehackernews.com/2026/11/openrouter-drops-3-providers-after.html
- Together AI, security notice (21 Nov 2026) — kredensial bocor via coding agent, sampel prompt Jun–Jul diunduh
- Phala, preliminary incident report — server inference dikuasai penyerang, dikandung 8 Nov 2026; jendela paparan DeepSeek V4 Flash 0731, 21 Okt–8 Nov 2026
- Novita, privacy notice — masalah routing MiniMax
- Cyber Security News, "OpenRouter Users' Private AI Chats Possibly Exposed Via Compromised Third-Party Providers" (22 Nov 2026) — https://cybersecuritynews.com/openrouter-ai-chat-leaks/
