# Terawang — Worker Matrix Transposer

Daemon Python yang memantau folder FTP, mengelompokkan 8 berkas sinyal sensor suara menjadi satu batch, menghitung matriks kecepatan 8x8 dengan GCC-PHAT (numpy 2.3, scipy 1.15.3), lalu mengunggah hasil ke FTP dan mengirim notifikasi ke RabbitMQ (pika).

| | |
|---|---|
| Produk | Terawang (Sound Sensor) |
| Layer | Worker (Python) |
| PIC | Arsal |
| Asal | Cermin fork dari `ASPRG/terawang-matrix-transposer`; pekerjaan aktif masih di repo induk sampai issue selesai, lalu disinkronkan ke INA |

## Repo terkait

| Repo | Peran |
|---|---|
| `iot-terawang` | Firmware sensor suara (sumber sinyal) |
| `worker-terawang` | Worker Node.js produk Terawang |
| `worker-terawang-dashboard` | API + halaman dashboard survei (Flask) |
| `be-terawang` | Backend |
| `web-terawang` | Web |
| `mobile-terawang` | Aplikasi mobile |

## Alur data

```
FTP FTP_SOURCE_FOLDER  <guid>_<timestamp>_<idx>.json (idx 1..8)
   --> index.py (poll tiap 10 detik, satu batch per siklus)
   --> FTP FTP_FOLDER_HASIL     SURVEY-<uuid4>-2025.json (larik 8x8)
   --> FTP FTP_FOLDER_DATA_ROW  salinan 8 berkas sumber
   --> RabbitMQ  result_queue   nama berkas hasil (teks)
   --> RabbitMQ  graph_row      {"GUID_SURVEY": "...", "data": [nama 8 berkas]}
```

- Batch valid: 8 berkas dengan `guid` sama, indeks 1..8, selisih timestamp paling lama 15 menit (`GROUPING_TIME_WINDOW_MINUTES`, `index.py:23`).
- Berkas sumber dihapus dari FTP asal setelah hasil terunggah; salinan hasil disimpan lokal di `hasil_proses_lokal/`.
- Antrean dideklarasikan non-durable (`index.py:352`); setiap publish membuka satu koneksi baru.
- Diameter model tetap 0.3 (`DIAMETER`, `index.py:22`); nilai `inf` pada matriks diganti 0.

## Isi repo

| Berkas | Fungsi |
|---|---|
| `index.py` | Daemon produksi (`main()`, `process_batch`, `gcc`, `onetap`) |
| `TerawangCC.py`, `experimentTRYTERAWANGCC.py` | Skrip uji coba offline GCC-PHAT |
| `data/ketuk1..8.json` | Contoh sinyal 8 kanal untuk skrip uji coba |
| `hasil_velo_akhir.json` | Contoh keluaran matriks 8x8 |

## Konfigurasi

Dibaca dari `.env` lewat python-dotenv (`index.py:28-43`).

| Key | Isi |
|---|---|
| `FTP_HOST`, `FTP_PORT` | Alamat server FTP |
| `FTP_USER`, `FTP_PASSWORD` | Akun FTP |
| `FTP_SOURCE_FOLDER` | Folder sumber berkas sinyal (default `/terawang`) |
| `FTP_FOLDER_HASIL` | Folder hasil (default `/result`) |
| `FTP_FOLDER_DATA_ROW` | Folder salinan sumber (default `/data_row`) |
| `RABBITMQ_HOST`, `RABBITMQ_PORT` | Alamat broker |
| `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD` | Akun broker |
| `RABBITMQ_VHOST` | Virtual host |
| `RABBITMQ_QUEUE_RESULT` | Antrean hasil (default `result_queue`) |
| `RABBITMQ_GRAPH_QUEUE` | Antrean data grafik (default `graph_row`) |

Lihat `.env.example`.

## Menjalankan

```
python -m venv venv
pip install -r requirements.txt
python index.py
```

`requirements.txt`: `numpy==2.3`, `scipy==1.15.3`, `pika`, `python-dotenv`. Versi Python tidak dinyatakan; `match` (`index.py:182`) mensyaratkan 3.10 atau lebih baru.

## Catatan

- Default host, akun, dan password FTP/RabbitMQ tertanam di kode sebagai nilai bawaan `os.getenv` (`index.py:28-41`); tanpa `.env` daemon tetap berjalan dengan nilai tersebut.
- `TerawangCC.py` mengimpor `pandas` yang tidak ada di `requirements.txt` dan menulis ke `.//ToF_Worker//contoh.json` yang tidak ada di repo.
- README lama menyebut folder `data_mentah`; kode produksi membaca dari FTP, bukan folder lokal.
- Tidak ada Dockerfile, test, atau lockfile.
- Firmware `iot-terawang` mengunggah CSV `R<n>_<millis>.csv`, sedangkan worker membaca `<guid>_<timestamp>_<idx>.json`; penerbit berkas JSON belum diverifikasi.
