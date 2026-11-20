# ESP32 Gate Schneider PM5350

Firmware ESP32 untuk membaca data kelistrikan dan energi dari power meter **Schneider PM5350** (keluarga PM5300) lewat **RS485 / Modbus RTU**, lalu menyajikannya lewat **Ethernet (W5500)** sebagai halaman web dan endpoint JSON. Dipasang di MDP dan dibaca oleh dashboard di mini PC.

```
PM5350 --RS485/Modbus RTU--> ESP32 --SPI--> W5500 --RJ45--> Mini PC (dashboard)
```

Versi saat ini: **v6.1**

## Fitur

- Membaca arus (Ia, Ib, Ic, In, Iavg), unbalance arus, tegangan L-N dan L-L, unbalance tegangan, daya aktif dan reaktif per fasa dan total, serta frekuensi
- Membaca THD arus, THD tegangan L-N, dan THD tegangan L-L
- Membaca akumulasi energi Wh, VARh, dan VAh (Del, Rec, Del+Rec, Del-Rec)
- Web server di port 80: halaman monitor (`/`) dan data JSON (`/data`)
- Output JSON ke Serial USB setiap siklus (1 detik)
- Pembacaan Modbus yang tahan gangguan:
  - satu request gabungan per kelompok data, dengan cadangan otomatis ke blok terpisah jika ditolak meter
  - carrier sense, karena bus RS485 bisa dipakai master lain (mis. LoRa)
  - validasi CRC, panjang frame, dan kewajaran nilai
  - data basi otomatis ditampilkan `null`
- Ketahanan (v6.1):
  - Task Watchdog untuk core 0 (Modbus) dan core 1 (web)
  - pencatatan alasan reset (power-on, watchdog, brownout, crash)
  - pemulihan W5500 otomatis dan penutupan socket yang macet

## Perangkat keras

| Komponen | Keterangan |
|---|---|
| ESP32 DevKit (esp32dev) | Mikrokontroler utama |
| Modul W5500 | Ethernet lewat SPI (VSPI) |
| Konverter TTL-RS485 | Tipe auto-direction (tanpa pin DE/RE), sisi A dan B ke meter |
| Schneider PM5350 | Port RS485 aktif |

### Pin

| Fungsi | GPIO |
|---|---|
| W5500 SCK | 18 |
| W5500 MISO | 19 |
| W5500 MOSI | 23 |
| W5500 CS | 5 |
| W5500 RST | 4 |
| RS485 RX (Serial2) | 17 |
| RS485 TX (Serial2) | 16 |

Pin RX dan TX bersilang: GPIO17 ke pin TXD modul RS485, GPIO16 ke pin RXD modul RS485.

Jika memakai modul MAX485 manual (dengan pin DE dan RE), set `USE_DE_RE_PIN` ke `1` dan hubungkan DE+RE ke GPIO25. Jangan memakai GPIO4 karena dipakai RST W5500.

## Pengaturan PM5350

Pengaturan komunikasi di meter harus cocok dengan firmware:

| Parameter | Nilai |
|---|---|
| Protokol | Modbus RTU |
| Slave ID | 1 |
| Baud rate | 19200 |
| Parity | Even, 8 data bit, 1 stop bit (8E1) |

Ubah `PM_SLAVE_ID`, `PM_BAUD`, dan `PM_CONFIG` di `main.cpp` jika pengaturan meter berbeda.

## Pengaturan jaringan

Alamat default di `main.cpp`:

| Parameter | Nilai |
|---|---|
| IP ESP32 | 192.168.10.50 |
| Subnet | 255.255.255.0 |
| Gateway dan DNS | 192.168.10.10 |
| MAC | 02:53:50:32:00:01 |

Atur IP statis komputer atau mini PC di subnet yang sama (192.168.10.x). Sesuaikan `ip`, `gateway`, `dnsServer`, dan `mac` dengan jaringanmu. Kalau ada beberapa ESP32 di satu jaringan, MAC masing-masing harus berbeda.

## Build dan upload

Memakai [PlatformIO](https://platformio.org/). `platformio.ini`:

```ini
[env:esp32dev]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
board = esp32dev
framework = arduino

monitor_speed = 115200
monitor_filters = time, esp32_exception_decoder

upload_port = COM5
monitor_port = COM5

lib_deps =
    arduino-libraries/Ethernet@^2.0.2
```

1. Salin `main.cpp` ke folder `src/`.
2. Ubah `upload_port` dan `monitor_port` sesuai port komputermu.
3. Build dan upload dari PlatformIO.
4. Buka Serial Monitor di 115200 baud untuk melihat log dan JSON.

Kode memakai Arduino core 3.x (platform pioarduino). Kelas `ServerCompat` menutup celah kompatibilitas library Ethernet dengan core tersebut.

## Opsi kompilasi

Diatur lewat `#define` di bagian atas `main.cpp`:

| Opsi | Default | Fungsi |
|---|---|---|
| `USE_DE_RE_PIN` | 0 | 1 = pakai pin DE/RE manual |
| `SERIAL_JSON` | 1 | Kirim JSON ke Serial tiap siklus |
| `LOG_HTTP` | 0 | Cetak log tiap request web |
| `READ_THD` | 1 | Baca THD |
| `READ_ENERGY` | 1 | Baca akumulasi energi |
| `USE_WDT` | 1 | Aktifkan watchdog |
| `WDT_TIMEOUT_S` | 20 | Batas waktu watchdog (detik) |
| `ETH_IDLE_RESET_S` | 120 | Reset W5500 jika tidak ada klien web selama sekian detik (0 = mati) |
| `SOCK_STALE_MS` | 10000 | Batas socket dianggap macet |
| `MAIN_DEAD_RESTART_S` | 0 | Restart jika data utama tidak pernah sukses selama sekian detik (0 = mati) |

## Endpoint

| URL | Isi |
|---|---|
| `http://192.168.10.50/` | Halaman monitor |
| `http://192.168.10.50/data` | Data JSON |

### Kunci JSON utama

| Kelompok | Kunci |
|---|---|
| Arus (A) | `ia`, `ib`, `ic`, `in_curr`, `i_avg` |
| Unbalance arus (%) | `i_unb_a`, `i_unb_b`, `i_unb_c`, `i_unb` |
| Tegangan L-N (V) | `van`, `vbn`, `vcn`, `v_ln_avg` |
| Tegangan L-L (V) | `vab`, `vbc`, `vca`, `v_ll_avg` |
| Unbalance tegangan (%) | `v_unb_ab`, `v_unb_bc`, `v_unb_ca`, `v_unb_ll`, `v_unb_an`, `v_unb_bn`, `v_unb_cn`, `v_unb_ln` |
| Daya | `p1`, `p2`, `p3`, `p_total`, `q1`, `q2`, `q3`, `q_total` |
| Frekuensi (Hz) | `freq` |
| THD (%) | `thd_ia`, `thd_ib`, `thd_ic`, `thd_van`, `thd_vbn`, `thd_vcn`, `thd_uab`, `thd_ubc`, `thd_uca` |
| Energi aktif (Wh) | `ea_del`, `ea_rec`, `ea_sum`, `ea_dif` |
| Energi reaktif (VARh) | `er_del`, `er_rec`, `er_sum`, `er_dif` |
| Energi semu (VAh) | `es_del`, `es_rec`, `es_sum`, `es_dif` |

Nilai yang tidak tersedia atau basi dikirim sebagai `null`.

### Kunci diagnostik

| Kunci | Arti |
|---|---|
| `ok` | 1 jika data utama dan energi masih segar |
| `e_cur`, `e_pq`, `e_freq`, `e_thd`, `e_en` | Kode error terakhir per blok (0 = OK) |
| `pid` | Product ID meter |
| `req`, `fail`, `fb` | Total request, request gagal, burst dari master lain |
| `uptime_s` | Lama ESP32 menyala (detik) |
| `rst` | Alasan reset boot ini (1 power-on, 3 software, 4 crash, 6 task WDT, 9 brownout) |
| `boots`, `wdt`, `brown` | Jumlah boot, reset watchdog, dan brownout sejak power-on |
| `prev_up` | Uptime sesi sebelum reset (detik) |
| `eth_rst` | Berapa kali W5500 di-init ulang |
| `ldrop` | Berapa kali kabel Ethernet terdeteksi putus |
| `sk` | Socket macet yang ditutup paksa |
| `heap` | RAM bebas (byte) |
| `http_ago` | Detik sejak klien web terakhir dilayani (-1 = belum pernah) |
| `link` | Status kabel Ethernet (ON atau OFF) |

### Kode error Modbus

| Kode | Arti |
|---|---|
| 0x00 | OK |
| 0x01 - 0x04 | Exception dari meter |
| 0xE0 | ID slave salah |
| 0xE1 | Fungsi salah |
| 0xE2 | Timeout atau frame terpotong |
| 0xE3 | CRC salah |
| 0xE4 | Jumlah byte tidak cocok |
| 0xE5 | Ada byte tambahan (tabrakan di bus) |
| 0xE6 | Bus tidak pernah bebas |
| 0xE7 | Data tidak masuk akal (dibuang) |

## Pemecahan masalah

**Dashboard tidak bisa terhubung, tetapi ping berhasil**
Cek `/data` dari browser. Lihat nilai `http_ago`, `sk`, dan `eth_rst` di Serial Monitor. Socket W5500 yang macet biasanya terpulihkan otomatis dalam 2 menit.

**ESP32 sering restart**
Lihat baris `RESET:` saat boot. `BROWNOUT` berarti catu daya kurang kuat: pakai catu yang lebih besar dan tambahkan kapasitor dekat modul W5500. `TASK-WDT` berarti ada bagian kode yang hang.

**Semua nilai `null`, `e_cur` = 0xE2**
Tidak ada jawaban dari meter. Periksa kabel A/B, pengaturan baud dan parity di meter, slave ID, dan arah pin RX/TX (harus bersilang).

**Data kadang `null`, `e_cur` = 0xE5 atau 0xE3**
Ada gangguan atau tabrakan di bus RS485. Pastikan hanya ada satu master aktif atau master kedua sudah dikenali (lihat nilai `fb`).

**`Ethernet.h` tidak ditemukan di VS Code**
Jalankan Build agar PlatformIO mengunduh library, lalu pilih *PlatformIO: Rebuild C/C++ Project Index*.

## Pencatatan

Untuk menangkap jam kejadian putus komunikasi, jalankan skrip ini di komputer klien:

```powershell
while($true){
  $t = Get-Date -f "yyyy-MM-dd HH:mm:ss"
  $line = try {
    $r = Invoke-RestMethod http://192.168.10.50/data -TimeoutSec 3
    "$t OK up=$($r.uptime_s) rst=$($r.rst) boots=$($r.boots) wdt=$($r.wdt) eth_rst=$($r.eth_rst) sk=$($r.sk) ldrop=$($r.ldrop) heap=$($r.heap) ago=$($r.http_ago)"
  } catch { "$t GAGAL $($_.Exception.Message)" }
  Add-Content log_esp32.txt $line
  Start-Sleep 5
}
```

## Struktur repo

```
.
├── platformio.ini
├── README.md
└── src
    └── main.cpp
```
