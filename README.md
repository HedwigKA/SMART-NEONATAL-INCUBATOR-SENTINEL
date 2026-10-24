# Smart Neonatal Incubator Sentinel

Sistem pemantauan inkubator neonatus berbasis **Raspberry Pi 5**, **Flask REST API**, dan dashboard web untuk membantu perawat memantau kondisi mikroklimat serta aktivitas bayi secara real-time.

> **Catatan keselamatan:** proyek ini merupakan prototipe akademik untuk mata kuliah **Sistem Tertanam A – Informatika, Universitas Sanata Dharma**. Sistem tidak menggantikan perangkat medis tersertifikasi atau pengawasan tenaga kesehatan.

## Ringkasan

Smart Neonatal Incubator Sentinel menggabungkan sensor suhu, kelembapan, gerak, dan suara dengan mesin perangkat keras multithread. Sistem menggunakan finite-state machine (FSM) untuk menentukan status inkubator dan menampilkan peringatan pada dashboard.

Filosofi alarm perangkat:

- **Kondisi normal:** LED hijau menyala tetap.
- **Peringatan atau gangguan:** LED merah berkedip dan peringatan ditampilkan di dashboard.
- **Suhu/kelembapan di luar batas:** tidak membunyikan buzzer inkubator; peringatan diberikan melalui LED dan dashboard.
- **Inaktivitas bayi ≥ 2 menit, jadwal susu, atau CODE BLUE:** buzzer inkubator berbunyi.
- **Buzzer inkubator:** hanya dapat dimatikan dengan menahan tombol fisik di boks inkubator.

## Fitur

- Pemantauan suhu dan kelembapan menggunakan DHT11.
- Histeresis untuk mengurangi alarm yang hidup-mati terlalu cepat.
- Deteksi gerak menggunakan PIR HC-SR501.
- Alarm inaktivitas bayi setelah **120 detik** tanpa gerakan.
- Deteksi tangisan atau suara menggunakan sensor suara LM393.
- Pengingat jadwal susu melalui dashboard.
- Tombol **CODE BLUE** untuk mengaktifkan panggilan darurat.
- Indikator LED hijau, LED merah berkedip, dan buzzer.
- Dashboard web real-time dengan polling status setiap detik.
- Monitor audio opsional pada browser perawat.
- Token autentikasi opsional untuk endpoint kontrol.
- Mode simulasi untuk pengujian tanpa perangkat keras.
- Log kejadian klinis yang ditampilkan pada dashboard.

## Arsitektur

```text
Sensor GPIO / DHT11
        │
        ▼
Hardware Engine multithread
        │
        ▼
Finite-State Machine (FSM)
        │
        ├── LED dan buzzer
        └── Flask REST API
                │
                ▼
        Dashboard web perawat
```

## Struktur Proyek

| File | Keterangan |
| --- | --- |
| `app.py` | Backend Flask, finite-state machine, pembacaan sensor, kontrol GPIO, dan API. |
| `index.html` | Dashboard web perawat. File ini disajikan Flask dari folder `templates/`. |
| `requirements.txt` | Dependensi Python. |
| `Dokumentasi Teknis_ Smart Neonatal Incubator Sentinel.html` | Dokumentasi teknis dan penjelasan proyek. |

## Persyaratan

### Perangkat keras

- Raspberry Pi 5
- DHT11
- PIR HC-SR501
- Sensor suara LM393
- LED merah dan LED hijau
- Buzzer
- Push button fisik
- Resistor dan kabel jumper sesuai kebutuhan rangkaian

### Perangkat lunak

- Raspberry Pi OS atau sistem Linux yang kompatibel
- Python 3
- Flask
- GPIO Zero
- Adafruit CircuitPython DHT

## Konfigurasi GPIO

Konfigurasi bawaan pada `app.py` adalah sebagai berikut:

| Komponen | GPIO |
| --- | ---: |
| LED merah | 24 |
| LED hijau | 23 |
| Buzzer | 25 |
| Tombol fisik | 22 |
| PIR HC-SR501 | 17 |
| Sensor suara LM393 | 23* |
| DHT11 | Pin board `D4` |

> **Penting:** konfigurasi bawaan memiliki konflik pada `GPIO23`: pin tersebut digunakan oleh LED hijau dan sensor suara. Sebelum menjalankan mode hardware, ubah `Config.PIN_SOUND` ke GPIO yang tidak digunakan, misalnya `27`, lalu sesuaikan wiring. Program akan menghentikan proses jika mendeteksi konflik pin.

Polaritas modul dapat disesuaikan melalui konstanta berikut di `app.py`:

- `BUZZER_ACTIVE_HIGH`
- `PIR_ACTIVE_HIGH`
- `SOUND_ACTIVE_HIGH`

## Instalasi

```bash
git clone https://github.com/HedwigKA/SMART-NEONATAL-INCUBATOR-SENTINEL.git
cd SMART-NEONATAL-INCUBATOR-SENTINEL

python3 -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
mkdir -p templates
cp index.html templates/index.html
```

Pada Raspberry Pi, pastikan antarmuka GPIO dan koneksi sensor telah disiapkan sebelum menjalankan mode hardware.

## Menjalankan

### Mode simulasi tanpa hardware

Mode ini direkomendasikan untuk pengembangan dan pengujian dashboard:

```bash
INCUBATOR_SIM=1 python3 app.py
```

### Mode hardware Raspberry Pi

```bash
python3 app.py
```

Server berjalan pada port `5000` dan mendengarkan pada semua interface jaringan. Buka dashboard melalui:

```text
http://<alamat-ip-raspberry-pi>:5000
```

Contoh:

```text
http://192.168.1.20:5000
```

## Autentikasi API Kontrol

Secara default, endpoint kontrol tidak menggunakan token. Untuk mengaktifkan autentikasi, tetapkan environment variable `INCUBATOR_TOKEN` sebelum menjalankan aplikasi:

```bash
export INCUBATOR_TOKEN="ganti-dengan-token-yang-kuat"
python3 app.py
```

Dashboard akan meminta token ketika perawat pertama kali menjalankan perintah kontrol. Token disimpan hanya pada `sessionStorage` browser.

> Jalankan aplikasi dengan token pada jaringan yang tidak sepenuhnya dipercaya. Tanpa token, siapa pun yang dapat mengakses port aplikasi berpotensi mengirim perintah kontrol.

## Endpoint API

### Mendapatkan status sistem

```http
GET /api/status
```

Endpoint ini mengembalikan status FSM, pembacaan sensor, alarm aktif, konfigurasi ambang, status simulasi, dan log kejadian.

### Mengirim perintah kontrol

```http
POST /api/control
Content-Type: application/json
X-Auth-Token: <token-jika-diaktifkan>
```

Bangunkan bayi untuk jadwal susu:

```json
{"action": "wake_baby"}
```

Aktifkan CODE BLUE:

```json
{"action": "emergency_alert"}
```

Perintah untuk mematikan buzzer dari API ditolak oleh sistem. Buzzer hanya dapat dimatikan melalui tombol fisik di inkubator sesuai protokol keselamatan proyek.

## Status FSM

| Status | Makna |
| --- | --- |
| `IDLE_NORMAL` | Kondisi inkubator normal. |
| `CRYING_DISTRESS` | Sensor suara mendeteksi tangisan atau suara bayi. |
| `SENSOR_FAULT` | DHT11 tidak memberikan pembacaan yang valid. |
| `HUMIDITY_HAZARD` | Kelembapan berada di luar batas aman. |
| `THERMAL_HAZARD` | Suhu berada di luar batas aman. |
| `FEEDING_WAKE` | Buzzer aktif untuk stimulasi jadwal susu. |
| `INACTIVITY_ALERT` | Tidak ada gerakan selama minimal 2 menit. |
| `CODE_BLUE` | Panggilan darurat sedang aktif. |

Prioritas status darurat adalah **CODE BLUE**, inaktivitas, jadwal susu, suhu, kelembapan, gangguan sensor, kemudian tangisan.

## Ambang Bawaan

| Parameter | Nilai |
| --- | ---: |
| Suhu minimum | 27 °C |
| Suhu maksimum | 34 °C |
| Histeresis suhu | 1 °C |
| Kelembapan minimum | 50% RH |
| Kelembapan maksimum | 90% RH |
| Histeresis kelembapan | 5% RH |
| Peringatan awal inaktivitas | 90 detik |
| Alarm inaktivitas | 120 detik |
| Interval pembacaan DHT11 | 2 detik |
| Batas data DHT11 kedaluwarsa | 10 detik |

Nilai tersebut dapat disesuaikan di class `Config` pada `app.py`. Ambang suhu dan kelembapan tidak dapat diubah melalui dashboard.

## Pengujian

1. Jalankan mode simulasi:

   ```bash
   INCUBATOR_SIM=1 python3 app.py
   ```

2. Buka dashboard pada `http://localhost:5000`.
3. Pastikan status koneksi berubah menjadi **Terhubung**.
4. Uji tombol **BANGUNKAN BAYI** dan **TRIGGER CODE BLUE**.
5. Periksa perubahan status, indikator visual, log kejadian, dan respons API.
6. Untuk pengujian hardware, matikan daya terlebih dahulu sebelum mengubah wiring GPIO.

## Troubleshooting

### Dashboard menampilkan `KONEKSI TERPUTUS`

- Pastikan `app.py` sedang berjalan.
- Pastikan alamat IP dan port `5000` benar.
- Pastikan firewall tidak memblokir koneksi.
- Periksa log terminal aplikasi.

### Sistem masuk mode simulasi

Aplikasi otomatis beralih ke mode simulasi jika library GPIO tidak tersedia atau gagal diinisialisasi. Jalankan instalasi dependensi kembali dan pastikan aplikasi dijalankan pada Raspberry Pi untuk mode hardware.

### Error konflik GPIO

Periksa semua nilai `PIN_*` pada class `Config`. Setiap perangkat harus menggunakan GPIO yang berbeda, khususnya `PIN_SOUND` dan `PIN_LED_HIJAU` pada konfigurasi awal.

### DHT11 tidak merespons

- Periksa VCC, GND, dan jalur data.
- Pastikan `PIN_DHT` sesuai dengan koneksi board.
- Pastikan sensor tidak membaca nilai di luar rentang valid.
- Tunggu beberapa detik karena pembacaan DHT11 tidak dilakukan pada setiap iterasi loop.

## Lisensi dan Kontribusi

Proyek ini dibuat untuk kebutuhan pembelajaran. Perubahan pada wiring, ambang alarm, atau perilaku buzzer harus diuji terlebih dahulu dalam lingkungan aman sebelum digunakan pada perangkat fisik.

Untuk kontribusi, silakan buat branch terpisah, dokumentasikan perubahan, dan uji mode simulasi sebelum mengajukan pull request.
