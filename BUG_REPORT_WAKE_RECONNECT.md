# Laporan Bug & Analisis: Wake Reconnect Delay (AirportItlwm / hal_iwm)

## 1. Ringkasan Eksekutif
- **Target Hardware**: Intel Dual Band Wireless-AC 7265 (`hal_iwm`)
- **OS Version**: macOS Sequoia 15.8.2 (Build `24H232`)
- **Kext Target**: `AirportItlwm.kext` (Sequoia)
- **Gejala**: Setelah wake from sleep pada rentang waktu 16:02–16:03, koneksi ke SSID sebelumnya (`KANTOR LAMA`) membutuhkan waktu total **~25,8 detik** (Wake: `16:03:00.005` -> Terkoneksi: `16:03:25.885`).
- **Kesimpulan Utama**: Delay bukan disebabkan oleh hang atau loop pada driver, melainkan kombinasi dari:
  1. **Awal Wake**: Scan request pertama dari CoreWiFi langsung di-reject oleh driver dengan return code `0xe00002bc` (`kIOReturnError`), memicu macOS backoff timer ~5 detik.
  2. **Percobaan Asosiasi Pertama**: Sinyal AP sangat marjinal (**RSSI -82 dBm**) sehingga frame Auth/Assoc tidak direspons oleh AP di udara, menyebabkan macOS Association Timeout selama **10 detik penuh**.
  3. **Percobaan Kedua**: Setelah sinyal membaik (**RSSI -77 dBm**), proses join selesai sukses hanya dalam **292 ms**.

---

## 2. Timeline Kronologis Presisi (Unified Log)

Berdasarkan ekstraksi unified system log (`airportd`, `IO80211Family`, CoreWiFi) pada interface `en1`:

| Waktu (HH:MM:SS.mmm) | Komponen / Subsystem | Event / Log Message | Dampak |
| :--- | :--- | :--- | :--- |
| **16:03:00.005** | `airportd` / PM | System wake event received (`user_wake`). Interface `en1` mulai transisi D0. | Sistem bangun dari sleep. |
| **16:03:00.267** | CoreWiFi | `AUTO-JOIN: Auto-join STARTED (uuid=FDEDB, trigger=user_wake)` | CoreWiFi memicu auto-join cepat setelah wake. |
| **16:03:00.294** | CoreWiFi | Request scan ke 13 channel aktif. | Memulai pencarian SSID target. |
| **16:03:00.318** | `IO80211Family` | `Apple80211IOCTLSetWrapper: ifname['en1'] IOUC type 10/'APPLE80211_IOC_SCAN_REQ', len[5528] return -536870212/0xe00002bc` | **FAILED (Issue 1)**: Driver me-reject IOCTL scan karena hardware/firmware belum tuntas inisialisasi wake. |
| **16:03:00.493** | CoreWiFi | `AUTO-JOIN: Scan FAILED (duration=199ms, results=0, error=(-536870212 '(null)'))` | Scan cepat wake gagal. |
| **16:03:00.525 – 01.926** | CoreWiFi | Mencoba scan kanal deferred/cache, 0 match ditemukan. | Menjadwalkan retry auto-join dalam ~4,5 detik (`schedule=ac`). |
| **16:03:06.426** | CoreWiFi | `AUTO-JOIN: Auto-join STARTED (trigger=retry (ac))` | Retry scan otomatis dieksekusi. |
| **16:03:08.587** | CoreWiFi | `Found matching network (ssid=KANTOR LAMA, channel=2g11/20, rssi=-82)` | AP ditemukan pada channel 11 dengan sinyal lemah (-82 dBm). |
| **16:03:08.653** | `airportd` | Memanggil `Apple80211Associate2` ke SSID `KANTOR LAMA`. | Memulai proses 802.11 Authentication & Association. |
| **16:03:08.653 – 18.919** | Over-The-Air | **Hening selama 10,266 detik**. AP tidak membalas / frame ACK drop. | Driver menunggu balasan frame dari AP di udara. |
| **16:03:18.919** | `airportd` / CoreWiFi | `[DISASSOC] Will disassociate due to failed association request (DISASSOC_REASON = 10)`<br>`Adding KANTOR LAMA to list of networks blacklisted for scan offload, reason: Association Timeout.`<br>`Association failed for network "KANTOR LAMA" on interface en1, returned error Error Domain=com.apple.wifi.apple80211API.error Code=-3905 "tmpErr"` | **FAILED (Issue 2)**: Timer asosiasi macOS (10 detik) habis tanpa respon dari AP. |
| **16:03:23.453** | CoreWiFi | Menjadwalkan retry auto-join berikutnya (`trigger=retry (ac)`). | Backoff retry pasca kegagalan asosiasi. |
| **16:03:25.578** | CoreWiFi | Rescan channel 11 berhasil: `rssi=-77` (menguat +5 dBm). | Sinyal AP lebih memadai. |
| **16:03:25.657** | `airportd` | Percobaan asosiasi kedua via `Apple80211Associate2`. | Request Auth & Assoc dikirim ulang. |
| **16:03:25.885** | CoreWiFi | `AUTO-JOIN: Join SUCCEEDED (duration=292ms, network=KANTOR LAMA, channel=2g11/20, rssi=-77)` | **SUKSES**: Terkoneksi sempurna hanya dalam 292 milidetik. |

---

## 3. Analisis Akar Masalah (Root Causes)

### Akar Masalah 1: Scan Rejection `0xe00002bc` di Awal Wake (Penundaan ~5 Detik)
- **Lokasi Kode**: [`AirportItlwm/WCL/WCL.cpp`](file:///volumes/dokumen/backup/Repositories/itlwm/AirportItlwm/WCL/WCL.cpp) & [`itlwm/hal_iwm/scan.cpp`](file:///volumes/dokumen/backup/Repositories/itlwm/itlwm/hal_iwm/scan.cpp)
- **Mekanisme**:
  - Pada saat wake (`16:03:00.005`), macOS langsung memicu `AUTO-JOIN` dalam waktu 262 ms (`16:03:00.267`).
  - CoreWiFi mengirim IOCTL `APPLE80211_IOC_SCAN_REQ`.
  - Pada saat tersebut, subsistem hardware PCI / firmware Intel 7265 sedang dalam proses D0 resume (`iwm_init` / `iwm_mvm_up`).
  - Driver belum siap menerima command scan atau scan context masih dalam status reset, sehingga IOCTL langsung mengembalikan error `kIOReturnError` (`0xe00002bc`).
  - Karena scan pertama gagal total (0 network), CoreWiFi mengaktifkan interval backoff default ~5 detik sebelum mencoba scan kedua.

### Akar Masalah 2: Association Timeout Over-The-Air pada RSSI -82 dBm (Penundaan ~10 Detik)
- **Lokasi Log**: `airportd` (`16:03:08.653` s/d `16:03:18.919`)
- **Mekanisme**:
  - Pada percobaan scan kedua, AP `KANTOR LAMA` di channel 11 terdeteksi dengan sinyal marginal **-82 dBm**.
  - Driver mengirim paket 802.11 Authentication/Association. Namun karena SNR sangat rendah dan noise di band 2.4 GHz, AP tidak mengirimkan ACK / respon.
  - macOS `IO80211Family` memiliki hard timeout sebesar **10 detik** untuk proses asosiasi. Selama 10 detik ini, stack Wi-Fi terblokir menunggu balasan.
  - Setelah 10 detik habis, macOS mencatat `Association Timeout (tmpErr -3905)` dan memasukkan SSID ke blacklist sementara.
  - Begitu retry berikutnya berjalan di `16:03:25` dengan sinyal naik ke **-77 dBm**, paket transmisi berhasil di-ACK oleh AP dan koneksi instan terbangun dalam 292 ms.

---

## 4. Evaluasi terhadap Perubahan Kode Sebelumnya

Modifikasi yang telah diterapkan pada:
- [`itlwm/hal_iwm/mac80211.cpp`](file:///volumes/dokumen/backup/Repositories/itlwm/itlwm/hal_iwm/mac80211.cpp)
- [`itlwm/hal_iwm/scan.cpp`](file:///volumes/dokumen/backup/Repositories/itlwm/itlwm/hal_iwm/scan.cpp)
- [`itl80211/openbsd/net80211/ieee80211_node.c`](file:///volumes/dokumen/backup/Repositories/itlwm/itl80211/openbsd/net80211/ieee80211_node.c)
- [`AirportItlwm/WCL/WCL.cpp`](file:///volumes/dokumen/backup/Repositories/itlwm/AirportItlwm/WCL/WCL.cpp)

**Hasil Evaluasi**:
- **Positif**: Tidak ada kebocoran state, lockup kernel, atau thread freeze. Driver merespons semua perintah dan menyelesaikan siklus wake tanpa panic.
- **Celah yang Tersisa**: Driver belum memiliki mekanisme *gating/synchronization* untuk menahan (sleep-wait singkat / serialize) IOCTL scan yang datang terlalu cepat saat transisi wake masih berlangsung, sehingga melempar `kIOReturnError` langsung ke OS alih-alih menahan sejenak hingga firmware siap.

---

## 5. Rekomendasi Solusi untuk Percakapan Baru

### Rekomendasi 1: Penanganan `APPLE80211_IOC_SCAN_REQ` saat Wake
- Di `WCL::apple80211_ioctl`, pada handler `APPLE80211_IOC_SCAN_REQ`:
  - Periksa apakah driver sedang dalam fase transisi wake / card init.
  - Jika ya, jangan langsung mengembalikan error `0xe00002bc`. Lakukan tunggu terukur (misal `IOSleep` bertahap atau tunggu event gate hingga 300–500 ms) sampai `pScanCtx` dan firmware siap menerima scan command.
  - Dengan demikian, scan pertama pada `16:03:00` akan sukses dan memotong delay ~5 detik.

### Rekomendasi 2: Rate Fallback saat Transmisi Auth/Assoc di RSSI Rendah
- Di `hal_iwm/mac80211.cpp` pada pengiriman frame management (`IEEE80211_FC0_TYPE_MGT`):
  - Pastikan frame Authentication dan Association request selalu dikirimkan menggunakan Basic Rate terendah (1 Mbps / 2 Mbps pada band 2.4 GHz) dengan daya transmisi penuh (TX power max), bukan modulasi MCS tinggi.
  - Hal ini meminimalisir kemungkinan frame hilang di udara saat sinyal berada pada batas kritis (-82 dBm).

---

## 6. Referensi File Log Mentah
- File Log Tersimpan: `/Users/ays/.gemini/antigravity-cli/brain/d35a9440-2014-437f-9027-b2b95ed6ea71/.system_generated/steps/203/output.txt`
- Rentang Ekstraksi: `2026-11-07 16:02:50.000` s/d `2026-11-07 16:03:35.000`
- Subsystem: `com.apple.WiFiAgent`, `com.apple.wifi.CoreWiFi`, `com.apple.airportd`, `IO80211Family`
