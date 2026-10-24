# Smart Neonatal Incubator Sentinel

Proyek UTS mata kuliah **Sistem Tertanam A** — Informatika, Universitas Sanata Dharma.

Sistem pemantau inkubator bayi berbasis **Raspberry Pi 5** dengan backend **Flask REST API** dan mesin perangkat keras multithread, serta dashboard web untuk perawat.

## Fitur Utama
- Pemantauan suhu & kelembapan (DHT11) dengan histeresis.
- Deteksi gerak (PIR HC-SR501) dan alarm inaktivitas bayi (≥ 2 menit).
- Deteksi tangisan (sensor suara LM393).
- Pengingat jadwal susu dan mode darurat **CODE BLUE**.
- Indikator LED (hijau = aman, merah berkedip = peringatan) dan buzzer yang hanya bisa dimatikan via tombol fisik.
- Dashboard web real-time + token autentikasi opsional untuk API kontrol.
- Mode simulasi untuk pengujian tanpa hardware.

## Struktur
| File | Keterangan |
|------|------------|
| `app.py` | Backend Flask + FSM + driver GPIO |
| `index.html` | Dashboard web (letakkan di folder `templates/` saat dijalankan) |
| `Dokumentasi Teknis_ Smart Neonatal Incubator Sentinel.html` | Dokumentasi teknis lengkap |

## Menjalankan
```bash
pip install -r requirements.txt
mkdir -p templates && cp index.html templates/

# Mode simulasi (tanpa hardware)
INCUBATOR_SIM=1 python3 app.py

# Opsional: wajibkan token untuk API kontrol
export INCUBATOR_TOKEN="ganti-dengan-token-anda"
python3 app.py
```
Buka `http://<ip-raspberry-pi>:5000` di browser.

## Endpoint API
- `GET /api/status` — status sistem & sensor
- `POST /api/control` — perintah kontrol (`{"action": "<nama>"}`, header `X-Auth-Token` bila token di-set)

> Lihat file dokumentasi teknis untuk detail wiring, FSM, dan batasan sistem.
