# Pendahuluan

Project ini (`worker-comfyui`) menyediakan cara untuk menjalankan [ComfyUI](https://github.com/comfyanonymous/ComfyUI) sebagai worker API serverless di platform [RunPod](https://www.runpod.io/). Tujuan utamanya adalah memungkinkan user mengirim workflow generate gambar ComfyUI melalui satu panggilan API sederhana, lalu menerima hasil gambar, baik secara langsung sebagai string base64 maupun melalui upload ke bucket AWS S3.

Project ini memaketkan ComfyUI ke dalam image Docker, mengelola penanganan job melalui SDK `runpod`, memakai websocket untuk komunikasi yang efisien dengan ComfyUI, dan memudahkan konfigurasi lewat variabel lingkungan.

# Konvensi dan Aturan Project

Dokumen ini menjelaskan konvensi operasional dan struktural utama untuk project `worker-comfyui`. Walaupun saat ini belum ada aturan code-style yang ketat yang dipaksa oleh linter, mengikuti konvensi ini membantu menjaga konsistensi dan kelancaran development/deployment.

## 1. Konfigurasi

- **Variabel Lingkungan:** Semua konfigurasi eksternal (misalnya kredensial AWS S3, modifikasi perilaku RunPod seperti `REFRESH_WORKER`) **wajib** dikelola lewat variabel lingkungan.
- Lihat bagian "Config" dan "Upload image to AWS S3" di `README.md` untuk detail variabel yang tersedia.

## 2. Penggunaan Docker

- **Berbasis Container:** Development, testing, dan deployment sangat bergantung pada Docker.
- **Platform:** Saat build image Docker yang ditujukan untuk RunPod, **selalu** gunakan flag `--platform linux/amd64` untuk memastikan kompatibilitas.
  ```bash
  # Contoh perintah build
  docker build --platform linux/amd64 -t my-image:tag .
  ```
- **Build untuk Development:** Untuk iterasi development yang lebih cepat, gunakan `MODEL_TYPE=base` untuk melewati download model eksternal:
  ```bash
  docker build --build-arg MODEL_TYPE=base -t runpod/worker-comfyui:dev .
  ```
- **Kustomisasi:** Ikuti metode di `README.md` untuk menambahkan model/node kustom (Network Volume atau edit Dockerfile + snapshots).

## 3. Interaksi API

- **Struktur Input:** Panggilan API ke endpoint `/run` atau `/runsync` harus mengikuti struktur JSON yang dijelaskan di `README.md` ("API specification"). Key utamanya adalah `input`, berisi `workflow` (wajib, object) dan `images` (opsional, array).
- **Encoding Gambar:** Gambar input yang dikirim lewat `input.images` harus berupa string base64 (opsional termasuk prefix `data:[<mediatype>];base64,`).
- **Format Workflow:** Object `input.workflow` sebaiknya berisi JSON yang diekspor dari ComfyUI menggunakan opsi "Save (API Format)" (membutuhkan mengaktifkan "Dev mode Options" di pengaturan ComfyUI).
- **Struktur Output:** Response sukses berisi field `output.images`, yaitu **list of dictionaries**. Setiap dictionary berisi `filename` (string), `type` (`"s3_url"` atau `"base64"`), dan `data` (string berisi URL atau data base64). Lihat contoh API di `README.md` untuk struktur persisnya.
- **Komunikasi Internal:** Monitoring status job menggunakan ComfyUI websocket API (bukan HTTP polling) demi efisiensi.

## 4. Penanganan Error

- **Error yang Ramah User:** Selalu tampilkan pesan error yang bermakna kepada user, bukan error HTTP generik atau exception internal.
- **Integrasi ComfyUI:** Saat ComfyUI mengembalikan validation error, parse body response untuk mengambil detail error dan tampilkan dalam format terstruktur dan actionable.
- **Konteks yang Membantu:** Jika memungkinkan, berikan informasi opsi yang tersedia (misalnya model yang tersedia, parameter valid) agar user bisa memperbaiki request.
- **Fallback yang Graceful:** Penanganan error harus turun secara wajar—jika parsing detail error gagal, tampilkan raw response daripada menyembunyikan error sama sekali.

## 5. Alur Development

- **Perubahan Kode:** Setelah memodifikasi kode handler, selalu rebuild image Docker sebelum testing dengan `docker-compose`:
  ```bash
  docker-compose down
  docker build --build-arg MODEL_TYPE=base -t runpod/worker-comfyui:dev .
  docker-compose up -d
  ```
- **Debugging:** Gunakan logging/print statement secara strategis untuk memahami response API eksternal (seperti format error ComfyUI) sebelum mengimplementasikan error handling.
- **Testing:** Uji skenario error sebaik skenario sukses untuk memastikan UX yang bagus.

## 6. Pengujian

- **Unit Test:** Tes otomatis ada di direktori `tests/` dan dijalankan dengan `python -m unittest discover`. Tambahkan tes baru untuk fitur baru atau bug fix.
- **Lingkungan Lokal:** Gunakan `docker-compose up` untuk pengujian end-to-end lokal. Ini membutuhkan environment Docker yang terkonfigurasi dengan benar serta dukungan NVIDIA GPU.

## 7. Dependensi

- **Python:** Kelola dependensi Python menggunakan `pip` (atau `uv`) dan file `requirements.txt`. Jaga file ini tetap up-to-date.

## 8. Code Style (Panduan Umum)

- Walaupun tidak dipaksa oleh tooling, usahakan code jelas dan konsisten. Ikuti best practice Python (misalnya PEP 8).
- Gunakan nama variabel dan fungsi yang bermakna.
- Tambahkan komentar jika logika tidak jelas.

### **Deteksi Model Type**

Model dikategorikan berdasarkan tipe node menggunakan mapping berikut:

- `UpscaleModelLoader` → `upscale_models`
- `VAELoader` → `vae`
- `UNETLoader`, `UnetLoaderGGUF`, `Hy3DModelLoader` → `diffusion_models`
- `DualCLIPLoader`, `TripleCLIPLoader` → `text_encoders`
- `LoraLoader` → `loras`
- Dan loader spesialis tambahan untuk pengkategorian model yang tepat

## Dependensi Custom Node

Saat memperluas base image dengan custom node, beberapa node bisa membutuhkan versi dependensi tertentu agar berfungsi dengan benar.

### **Masalah Kompatibilitas yang Diketahui**

- **Masalah dependensi ComfyUI-BrushNet:** Membutuhkan versi dependensi tertentu: `diffusers>=0.29.0`, `accelerate>=0.29.0,<0.32.0`, dan `peft>=0.7.0` untuk mengatasi error import
- **Pola perbaikan:** Saat menemukan error import dari custom node, cek rantai dependensi dan pastikan versi yang kompatibel terinstall di Dockerfile menggunakan `uv pip install`

