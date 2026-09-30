# Trading Agents Lite

[English](README.md) | **Bahasa Indonesia**

Versi lite dari [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents) yang dibangun ulang sebagai **Claude Agent Skill**. Skill ini menjalankan alur riset multi-agen yang sama (tim analis, debat bull vs bear, trader, komite risiko, dan portfolio manager) langsung di dalam Claude, sehingga bisa dipakai dengan langganan Claude tanpa perlu API key terpisah.

> Ini adalah alat riset dan belajar. Skill ini tidak memberikan saran keuangan, tidak mengeksekusi transaksi, dan tidak mengklaim bisa menghasilkan keuntungan.

---

## Daftar isi

- [Apa ini](#apa-ini)
- [Cara kerja](#cara-kerja)
- [Perbandingan versi lite dan versi asli](#perbandingan-versi-lite-dan-versi-asli)
- [Kelebihan](#kelebihan)
- [Kekurangan](#kekurangan)
- [Hal yang tidak diklaim proyek ini](#hal-yang-tidak-diklaim-proyek-ini)
- [Kebutuhan](#kebutuhan)
- [Instalasi](#instalasi)
- [Cara pakai](#cara-pakai)
- [Menjalankan script secara terpisah](#menjalankan-script-secara-terpisah)
- [Struktur repository](#struktur-repository)
- [Kredit](#kredit)
- [Disclaimer](#disclaimer)

---

## Apa ini

TradingAgents versi asli adalah framework Python yang dibangun di atas LangGraph. Setiap agen merupakan panggilan terpisah ke model bahasa melalui API, sehingga kamu butuh API key dari penyedia seperti OpenAI, Anthropic, atau Google, dan biayanya dihitung per token.

Trading Agents Lite mempertahankan **desain peran** dari versi asli lalu memindahkannya ke dalam skill Claude, yaitu kumpulan instruksi Markdown ditambah beberapa script Python kecil. Claude membaca instruksi tersebut dan menjalankan setiap peran, sedangkan script menangani bagian yang harus presisi (perhitungan indikator, pembulatan harga, ukuran posisi, dan memori keputusan).

Disebut "lite" karena beberapa bagian versi asli sengaja tidak disertakan: tidak ada graph engine, tidak ada dukungan banyak penyedia model, tidak ada pemulihan dari checkpoint, dan tidak bisa dijalankan massal lewat kode. Sebagai gantinya, skill ini hampir tidak butuh setup.

Proyek ini adalah implementasi ulang yang independen. Tidak ada kode dari repository asli di dalamnya, dan proyek ini tidak berafiliasi dengan maupun didukung oleh Tauric Research atau Anthropic.

## Cara kerja

Satu analisis mencakup satu ticker pada satu tanggal analisis dan melewati tujuh tahap. Setiap peran menulis hasilnya ke file tersendiri di folder run, sehingga peran berikutnya membaca file, bukan mengandalkan apa yang sudah dibahas sebelumnya di percakapan.

1. **Data.** Snapshot data dibangun dari satu atau beberapa sumber: `fetch_data.py` (Yahoo Finance lewat `yfinance`, jika jaringan mengizinkan), file CSV harga yang kamu upload, atau pencarian web. Data yang bertanggal setelah tanggal analisis tidak dipakai. Indikator dihitung oleh `indicators.py`, bukan ditebak oleh model.
2. **Tim analis.** Empat analis menulis laporan secara independen: Teknikal, Fundamental, Berita & Makro, dan Sentimen. Tidak ada analis yang melihat laporan analis lain.
3. **Tim riset.** Peneliti Bull dan Peneliti Bear berdebat selama 1 sampai 3 ronde. Setiap giliran wajib membantah poin terkuat lawan lebih dulu. Research Manager menilai debat lalu menyusun rencana investasi beserta rating.
4. **Trader.** Rencana diubah menjadi usulan transaksi yang konkret. `risk_tools.py` menghitung entry, stop, target, rasio reward terhadap risiko, dan ukuran posisi, lalu membulatkannya ke fraksi harga yang valid dan, untuk saham Indonesia, ke satuan lot.
5. **Komite risiko.** Anggota Agresif, Netral, dan Konservatif menguji usulan tersebut, dan masing-masing mengusulkan penyesuaian yang konkret.
6. **Portfolio Manager.** Menerima atau menolak setiap penyesuaian, menerapkan pelajaran dari keputusan sebelumnya, lalu memberi rating akhir dalam skala lima tingkat: Buy, Overweight, Hold, Underweight, Sell.
7. **Hasil.** Ringkasan singkat di chat, laporan lengkap dalam file Markdown, dan catatan di log keputusan (`trading_agents_log.jsonl`) yang berfungsi sebagai memori untuk analisis berikutnya.

Skill ini punya dua mode dan memilih secara otomatis:

- **Mode subagent (Claude Code, Cowork):** setiap peran berjalan sebagai subagent terpisah dengan konteksnya sendiri. Mode ini paling mendekati desain versi asli.
- **Mode satu konteks (chat claude.ai):** satu percakapan Claude menjalankan semua peran secara berurutan. Pemisahan dijaga lewat prosedur: setiap laporan hanya ditulis dari file input miliknya sendiri dan tidak direvisi setelah laporan lain muncul.

Kedalaman analisis bisa diatur: `quick`, `standard` (bawaan), atau `deep`. Semakin dalam, semakin panjang laporannya dan semakin banyak ronde debatnya.

![Alur agen Trading Agents Lite](assets/diagram.png)

## Perbandingan versi lite dan versi asli

| Aspek | TradingAgents (asli) | Trading Agents Lite |
|---|---|---|
| Bentuk | Framework Python di atas LangGraph, dengan CLI dan API Python | Skill Claude: instruksi Markdown ditambah script Python pendukung |
| Model bahasa | Banyak penyedia lewat API key, termasuk model lokal via Ollama | Hanya Claude, sesuai model Claude yang sedang kamu pakai |
| Model biaya | Bayar per token API (atau gratis dengan model lokal) | Terhitung dari kuota paket Claude, tanpa API key terpisah |
| Pemisahan agen | Setiap agen adalah panggilan model terpisah di dalam graph | Subagent terpisah di Claude Code; satu konteks bersama dengan pemisahan prosedural di claude.ai |
| Kombinasi model | Model berbeda untuk peran "berpikir mendalam" dan "berpikir cepat" | Satu model untuk semua peran |
| Data pasar | Penyedia data bawaan (Yahoo Finance, Alpha Vantage, dan lainnya di versi terbaru) | Script `yfinance` jika jaringan mengizinkan, selain itu CSV yang diupload atau pencarian web |
| Memori | Log keputusan otomatis dengan refleksi | `decision_log.py`; di claude.ai kamu sendiri yang menyimpan dan mengupload ulang file log |
| Pemulihan saat error | Melanjutkan dari checkpoint | Tidak tersedia |
| Otomatisasi | Bisa dijalankan lewat script untuk banyak ticker dan tanggal | Berbasis percakapan, satu analisis setiap kali |
| Setup | Install paket dan atur API key | Upload file zip (claude.ai) atau salin folder (Claude Code) |
| Aturan pasar | Mendukung semua pasar yang tersedia di Yahoo Finance | Juga menambahkan aturan Bursa Efek Indonesia secara eksplisit: fraksi harga, lot, catatan tentang batas auto rejection |

## Kelebihan

- **Tidak perlu API key.** Analisis berjalan di dalam Claude dan memakai paket langgananmu. Ini alasan utama versi lite dibuat.
- **Angka dihitung, bukan ditebak.** RSI, MACD, ADX, ATR, Bollinger Bands, moving average, level pivot, volatilitas, drawdown, dan beta berasal dari `indicators.py`. Agen diinstruksikan untuk menyalin angka dari output script atau sumber yang dikutip, dan menulis "tidak tersedia" daripada menebak.
- **Sumber bisa ditelusuri.** Temuan dari web dicatat di `sources.md` lengkap dengan tanggal, penerbit, dan URL, lalu dikutip di dalam laporan.
- **Perlindungan dari look-ahead bias.** Harga dipotong pada tanggal analisis, laporan keuangan kuartalan baru dipakai sekitar 45 hari setelah periodenya berakhir (sebagai perkiraan jeda publikasi), dan berita yang terbit setelah tanggal analisis dibuang. Karena itu analisis untuk tanggal di masa lalu tetap bisa dilakukan, dengan catatan yang dijelaskan di bagian kekurangan.
- **Rencana transaksi yang valid di bursa.** Stop dan target dibulatkan ke fraksi harga yang berlaku, ukuran posisi dibulatkan ke lot utuh di Bursa Efek Indonesia, dan besarnya posisi dihitung dari batas risiko.
- **Input data fleksibel.** Tetap bisa berjalan saat Yahoo Finance tidak bisa diakses. Parser CSV bisa membaca file ekspor dari Yahoo Finance, Investing.com (termasuk header dan format angka Indonesia seperti `9.875` untuk 9875), Nasdaq.com, TradingView, dan aplikasi broker.
- **Transparan.** Output setiap peran disimpan dan disusun menjadi laporan lengkap, sehingga kamu bisa melihat bagaimana rating akhir diputuskan dan di bagian mana kamu tidak setuju.
- **Ringan.** Tidak ada framework graph. Script hanya butuh `pandas` dan `numpy`; `yfinance` bersifat opsional.

## Kekurangan

Baca bagian ini sebelum mengandalkan hasil apa pun.

1. **Satu model menjalankan semua peran.** Semua agen memakai model Claude yang sama, sehingga kecenderungan cara berpikirnya juga sama. Di claude.ai, semua agen juga berbagi satu percakapan, jadi independensi antar agen bersifat prosedural, bukan struktural. Debat bull vs bear bisa jadi kurang tajam dibandingkan debat antar model yang dijalankan terpisah.
2. **Tidak bisa memilih model.** Kamu tidak bisa memakai model yang lebih kuat untuk peran pengambil keputusan dan model yang lebih murah untuk sisanya, seperti di versi asli. Kualitas hasil mengikuti model Claude yang kamu pakai.
3. **Data lebih terbatas di claude.ai.** Sandbox claude.ai memblokir Yahoo Finance secara default. Tanpa CSV yang diupload, analisis teknikal beralih ke data dari web dan hasilnya jelas kurang lengkap. Data fundamental banyak saham Indonesia di sumber gratis juga tidak merata. Analisis sentimen berupa pembacaan kualitatif atas hasil pencarian, bukan data sistematis dari StockTwits, Stockbit, atau X.
4. **Batas pemakaian.** Satu analisis standar menghasilkan banyak pesan panjang. Mode deep dan perbandingan beberapa ticker memakai kuota paket jauh lebih banyak dan bisa mencapai batas pemakaian.
5. **Hasil berbeda di setiap run.** Output model bahasa tidak deterministik. Ticker dan tanggal yang sama bisa menghasilkan laporan yang berbeda, dan sesekali rating yang berbeda.
6. **Belum divalidasi.** Belum ada backtest sistematis untuk skill ini. Script pendukung sudah diuji dengan data sintetis dalam beberapa format file, tetapi alur lengkapnya belum dievaluasi dengan data historis pasar yang sebenarnya. Nilai indikator bisa sedikit berbeda dari platform charting karena perbedaan metode smoothing dan nilai awal.
7. **Heuristik adalah penyederhanaan.** Skor tren, deteksi pivot, batas likuiditas, dan jeda publikasi 45 hari adalah aturan praktis, bukan model yang presisi.
8. **Perlindungan untuk tanggal lampau tidak sempurna.** Rasio profil perusahaan dari Yahoo adalah nilai saat ini, dan sumber web maupun media sosial tidak selalu bisa disaring sesuai apa yang sudah diketahui pada tanggal tersebut. Skill ini akan memberi tanda, tetapi mode backtest tetap lebih lemah dibandingkan database historis yang lengkap.
9. **Aturan pasar bisa berubah.** Tabel fraksi harga dan batas auto rejection di Bursa Efek Indonesia bisa berubah. Skill ini meminta Claude memeriksa aturan terbaru saat aturan tersebut berpengaruh.
10. **Tanpa otomatisasi, pemulihan, maupun eksekusi.** Tidak ada penjadwalan, tidak ada fitur melanjutkan dari checkpoint, dan tidak terhubung ke broker mana pun.
11. **Memori manual di claude.ai.** File di sandbox claude.ai tidak tersimpan antar chat. Untuk menyimpan riwayat keputusan, download `trading_agents_log.jsonl` setelah analisis lalu upload lagi di analisis berikutnya.

## Hal yang tidak diklaim proyek ini

- Proyek ini tidak mengklaim bisa mengalahkan pasar atau mengulang hasil return yang pernah dilaporkan untuk framework aslinya.
- Proyek ini tidak mengklaim ratingnya benar atau cocok untuk kondisi keuanganmu.
- Proyek ini tidak mengklaim setara dengan TradingAgents versi asli. Ini adalah adaptasi sederhana dari desain perannya.

## Kebutuhan

**Untuk claude.ai (web atau aplikasi desktop)**
- Paket Claude yang mendukung custom skill, dengan fitur code execution aktif. Lihat dokumentasi resmi tentang [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) dan [Claude Help Center](https://support.claude.com) untuk informasi paket dan pengaturan terbaru.

**Untuk Claude Code**
- Claude Code sudah terinstall, lihat [dokumentasi Claude Code](https://docs.claude.com/en/docs/claude-code/overview).
- Python 3.10 atau lebih baru dengan paket yang tercantum di `requirements.txt`.

## Instalasi

### 1. Clone repository

```bash
git clone https://github.com/<username-kamu>/trading-agents-lite.git
cd trading-agents-lite
```

Ganti `<username-kamu>` dengan akun GitHub tempat repository ini disimpan. Kamu juga bisa memakai tombol **Code > Download ZIP** di halaman GitHub lalu mengekstraknya.

### 2a. Pasang di claude.ai

1. Pastikan fitur code execution sudah aktif di pengaturan Claude.
2. Buat file zip yang berisi folder `trading-agents` di level paling atas (folder yang berisi `SKILL.md`):

   macOS atau Linux:
   ```bash
   zip -r trading-agents.zip trading-agents
   ```
   Windows PowerShell:
   ```powershell
   Compress-Archive -Path trading-agents -DestinationPath trading-agents.zip
   ```
3. Buka bagian Skills di pengaturan Claude lalu upload `trading-agents.zip`. Tergantung versi aplikasinya, menu ini ada di **Settings > Capabilities** atau **Customize > Skills**.
4. Pastikan skill dalam keadaan aktif.

### 2b. Pasang di Claude Code

Salin folder skill ke direktori skill pribadi (bisa dipakai di semua proyek):

```bash
mkdir -p ~/.claude/skills
cp -r trading-agents ~/.claude/skills/
pip install -r requirements.txt
```

Windows PowerShell:
```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse trading-agents "$HOME\.claude\skills\"
pip install -r requirements.txt
```

Kalau hanya ingin dipakai di satu proyek, salin ke `.claude/skills/` di dalam folder proyek tersebut. Disarankan memakai virtual environment untuk paket Python.

## Cara pakai

Buka chat baru lalu tulis permintaan seperti biasa. Skill akan aktif saat kamu meminta analisis atau penilaian saham maupun aset kripto.

```text
analisis saham BBCA
BBRI layak beli?
analisis mendalam TLKM, saya punya modal 50 juta
cek cepat BTC
bandingkan BBRI, BMRI, dan BBNI
analisis ASII per tanggal 1 Maret 2025
```

Permintaan dalam bahasa Inggris juga bisa, misalnya `Analyse NVDA`.

Opsi yang bisa kamu sebutkan di permintaan:

| Kamu menulis | Efeknya |
|---|---|
| "cepat" atau "quick" | Laporan lebih singkat, 1 ronde debat |
| "mendalam" atau "deep" | Laporan lebih panjang, 3 ronde debat, 2 ronde komite risiko |
| "teknikal dan fundamental saja" | Hanya menjalankan sebagian analis |
| "per tanggal 1 Maret 2025" | Menganalisis tanggal lampau (mode backtest) |
| Modal, harga rata-rata, atau profil risiko | Dipakai Trader dan Portfolio Manager untuk ukuran posisi dan arahan |

Format ticker mengikuti Yahoo Finance: `BBCA.JK` untuk saham Indonesia, `AAPL` untuk saham AS, `BTC-USD` untuk kripto, dan `0700.HK`, `7203.T`, dan seterusnya untuk pasar lain. Kode empat huruf tanpa akhiran dalam konteks Indonesia otomatis dianggap sebagai saham IDX.

### Supaya hasilnya lebih baik di claude.ai

- **Upload file harga harian 1 sampai 2 tahun** bersama permintaanmu. Dengan begitu analisis teknikal bisa dilakukan secara lengkap. File ekspor dari Investing.com, Yahoo Finance, TradingView, atau aplikasi broker semuanya bisa dipakai. File benchmark (misalnya IHSG) juga membuat perhitungan beta dan kekuatan relatif bisa dilakukan.
- **Atau izinkan host Yahoo Finance** di pengaturan jaringan jika paketmu menyediakan opsi tersebut. Saat diblokir, script akan menampilkan daftar host yang perlu diizinkan; `yfinance` memakai `query1.finance.yahoo.com`, `query2.finance.yahoo.com`, `fc.yahoo.com`, `finance.yahoo.com`, `guce.yahoo.com`, dan `consent.yahoo.com` (atau izinkan `*.yahoo.com` jika wildcard didukung).
- **Claude Code on the web / sesi cloud**: level jaringan bawaan **Trusted** memblokir Yahoo. Buka pemilih environment (ikon awan di atas kotak pesan di claude.ai/code), edit environment, ubah **Network access** menjadi **Custom**, tambahkan `*.yahoo.com` di **Allowed domains**, biarkan **Also include default list of common package managers** tercentang, simpan, lalu mulai sesi baru.
- **Simpan log keputusan.** Download `trading_agents_log.jsonl` setelah analisis, lalu upload bersama permintaan berikutnya supaya Portfolio Manager bisa meninjau keputusan sebelumnya.

### Yang kamu dapatkan

- Ringkasan di chat: rating akhir, tingkat keyakinan, horizon waktu, entry, stop, target, ukuran posisi, pandangan setiap meja analis, poin bull dan bear terkuat, serta kondisi yang bisa mengubah pandangan.
- Laporan lengkap `{TICKER}_{TANGGAL}_trading-agents.md` berisi output semua agen, snapshot data, dan daftar sumber.
- File `trading_agents_log.jsonl` yang sudah diperbarui.

## Menjalankan script secara terpisah

Script pendukung juga bisa dijalankan dari terminal tanpa Claude.

```bash
cd trading-agents/scripts

# Snapshot teknikal dari file CSV harga harian apa pun
python indicators.py --csv harga.csv --ticker BBCA.JK --date 2026-09-29 \
    --benchmark-csv ihsg.csv --md technical.md --out technical.json

# Ambil semua data dari Yahoo Finance (butuh akses internet ke Yahoo)
python fetch_data.py BBCA.JK --date 2026-09-29 --outdir runs/BBCA

# Stop, target, dan ukuran posisi sesuai fraksi harga
python risk_tools.py --ticker BBCA.JK --entry 9800 --atr 160 --stop-atr 2 \
    --targets-r 1.5,3 --equity 100000000 --risk-pct 1

# Memori keputusan
python decision_log.py add --ticker BBCA.JK --rating Overweight --price 9800 --thesis "..."
python decision_log.py review --ticker BBCA.JK --price 10150
```

Jalankan script apa pun dengan `--help` untuk melihat semua opsinya.

## Struktur repository

```text
trading-agents-lite/
├── README.md                 Dokumentasi bahasa Inggris
├── README.id.md              Dokumentasi bahasa Indonesia
├── requirements.txt          Paket Python untuk script pendukung
├── assets/
│   ├── agent-flow.svg        Diagram alur (bahasa Inggris)
│   ├── agent-flow.id.svg     Diagram alur (bahasa Indonesia)
│   └── diagram.png           Diagram alur detail (dibuat oleh gitdiagram.com)
└── trading-agents/           Skill-nya (folder inilah yang dipasang)
    ├── SKILL.md              Instruksi alur kerja dan orkestrasi
    ├── references/
    │   ├── agents.md         Deskripsi peran untuk 12 agen
    │   ├── data-sources.md   Jalur data, cadangan, resep pencarian, aturan pasar
    │   └── output-format.md  Template ringkasan chat dan laporan lengkap
    └── scripts/
        ├── common.py         Deteksi pasar, benchmark, fraksi harga dan lot IDX
        ├── fetch_data.py     Pengambilan data Yahoo Finance dengan filter look-ahead
        ├── indicators.py     Snapshot teknikal dari file OHLCV apa pun
        ├── risk_tools.py     Stop, target, rasio reward terhadap risiko, ukuran posisi
        └── decision_log.py   Memori keputusan dan peninjauan hasil
```

Instruksi di dalam skill ditulis dalam bahasa Inggris supaya lebih presisi. Claude tetap menjawab dalam bahasa yang kamu gunakan.

## Kredit

Peran agen, struktur debat, dan skala rating lima tingkat mengikuti desain TradingAgents dari Tauric Research. Jika kamu memakai proyek ini, mohon cantumkan juga kredit untuk karya aslinya:

```bibtex
@misc{xiao2025tradingagentsmultiagentsllmfinancial,
      title={TradingAgents: Multi-Agents LLM Financial Trading Framework},
      author={Yijia Xiao and Edward Sun and Di Luo and Wei Wang},
      year={2025},
      eprint={2412.20138},
      archivePrefix={arXiv},
      primaryClass={q-fin.TR},
      url={https://arxiv.org/abs/2412.20138},
}
```

## Disclaimer

Proyek ini dibuat untuk riset dan edukasi. Outputnya dihasilkan oleh model bahasa dari data yang bisa saja tidak lengkap, terlambat, atau keliru. Hasilnya bukan saran keuangan, investasi, maupun trading. Investasi di pasar mengandung risiko, termasuk kehilangan modal. Lakukan riset sendiri dan pertimbangkan untuk berkonsultasi dengan profesional berlisensi sebelum mengambil keputusan.
