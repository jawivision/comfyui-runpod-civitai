# Deployment

Panduan ini menjelaskan cara deploy `worker-comfyui` sebagai serverless endpoint di RunPod, mencakup penggunaan image official yang sudah jadi maupun image custom yang Anda build sendiri.

## Deploy Image Official (Pre-Built)

Ini adalah metode paling sederhana jika image official sudah memenuhi kebutuhan Anda.

### Buat Template (opsional)

- Buat [template baru](https://runpod.io/console/serverless/user/templates) dengan klik `New Template`.
- Pada dialog, atur:
  - Template Name: `worker-comfyui` (atau nama lain)
  - Template Type: serverless (ubah template type ke “serverless”)
  - Container Image: gunakan salah satu tag official, misalnya `runpod/worker-comfyui:<version>-sd3` (lihat daftar tag di `README.md`).
  - Container Registry Credentials: biarkan default (image public).
  - Container Disk: sesuaikan berdasarkan tag image yang dipilih (lihat [Rekomendasi GPU](#rekomendasi-gpu-untuk-image-official)).
  - (opsional) Environment Variables: konfigurasi S3 atau pengaturan lain (lihat `docs/configuration.md`).
    - Catatan: jika Anda tidak mengonfigurasi S3, gambar akan dikembalikan sebagai base64. Untuk storage persisten antar job tanpa S3, pertimbangkan memakai Network Volume (lihat `docs/customization.md`). Jika model dari network volume tidak terdeteksi, lihat `docs/network-volumes.md`.
- Klik `Save Template`.

### Buat Endpoint

- Masuk ke [`Serverless > Endpoints`](https://www.runpod.io/console/serverless/user/endpoints) lalu klik `New Endpoint`.
- Pada dialog, atur:
  - Endpoint Name: `comfy` (atau nama lain)
  - Worker configuration: pilih GPU yang mampu menjalankan model pada image tag yang Anda pilih (lihat [Rekomendasi GPU](#rekomendasi-gpu-untuk-image-official)).
  - Active Workers: `0` (skalakan sesuai beban)
  - Max Workers: `3` (atau sesuai limit budget)
  - GPUs/Worker: `1`
  - Idle Timeout: `5` (default biasanya cukup)
  - Flash Boot: `enabled` (direkomendasikan agar startup lebih cepat)
  - Select Template: pilih template yang Anda buat
  - (opsional) Advanced: jika memakai Network Volume, pilih pada `Select Network Volume` (lihat `docs/customization.md` dan `docs/network-volumes.md`)
- Klik `deploy`.

### Rekomendasi GPU (untuk Image Official)

| Model                     | Suffix Tag Image | Minimum VRAM | Rekomendasi Ukuran Disk Container |
| ------------------------- | ---------------- | ------------ | --------------------------------- |
| Stable Diffusion XL       | `sdxl`           | 8 GB         | 15 GB                             |
| Stable Diffusion 3 Medium | `sd3`            | 5 GB         | 20 GB                             |
| FLUX.1 Schnell            | `flux1-schnell`  | 24 GB        | 30 GB                             |
| FLUX.1 dev                | `flux1-dev`      | 24 GB        | 30 GB                             |
| Base (tanpa model)        | `base`           | N/A          | 5 GB                              |

Catatan: ukuran disk bersifat perkiraan dan dapat sedikit berubah. Image custom akan bervariasi sesuai model/node yang Anda masukkan.

## Deploy Setup Custom

Jika Anda sudah membuat environment custom (mis. menambah node, menambah model, atau mengubah Dockerfile), berikut cara deploy-nya.

> Ingin melewati setup manual?
>
> [ComfyUI-to-API](https://comfy.getrunpod.io) dapat membuat repo GitHub + Dockerfile custom dari workflow ComfyUI Anda secara otomatis. Setelah itu Anda bisa deploy lewat GitHub Integration di RunPod.

### Metode 1: Build Manual, Push, dan Deploy

1. **Tulis Dockerfile:** ikuti `docs/customization.md` untuk membuat Dockerfile yang berisi base image, nodes, models, dan file statis yang dibutuhkan.
2. **Build Docker image:** masuk ke folder yang berisi Dockerfile, lalu build:
   ```bash
   docker build --platform linux/amd64 -t <nama-image>:<tag> .
   ```
   - Selalu sertakan `--platform linux/amd64` agar kompatibel dengan RunPod.
3. **Tag untuk registry:** (contoh Docker Hub)
   ```bash
   docker tag <nama-image>:<tag> <username-registry>/<nama-image>:<tag>
   ```
4. **Login ke registry:**
   ```bash
   docker login
   ```
5. **Push image:**
   ```bash
   docker push <username-registry>/<nama-image>:<tag>
   ```
6. **Deploy di RunPod:**
   - Buat template seperti langkah di atas, namun isi `Container Image` dengan nama image yang sudah Anda push (mis. `<username>/<nama-image>:<tag>`).
   - Jika registry private, Anda perlu mengisi Container Registry Credentials.
   - Sesuaikan `Container Disk` dengan isi image custom Anda.
   - Buat endpoint menggunakan template tersebut.

### Metode 2: Deploy via RunPod GitHub Integration

RunPod dapat build dan deploy langsung dari repo GitHub yang berisi `Dockerfile`.

1. Pastikan repo GitHub Anda berisi `Dockerfile` (di root atau path yang Anda tentukan).
2. Hubungkan GitHub ke RunPod (otorisasi akses repo).
3. Buat endpoint baru dan pilih opsi “Start from GitHub Repo”.
4. Konfigurasikan:
   - repository dan branch
   - Context Path (biasanya `/` jika Dockerfile di root)
   - Dockerfile Path (biasanya `Dockerfile`)
   - resource GPU/workers
   - environment variables (lihat `docs/configuration.md`)
5. Deploy. Setiap `git push` ke branch yang dipilih akan memicu build ulang dan update endpoint.

## Tambahan (Kustom Repo Ini)

Bagian ini adalah tambahan untuk repo Anda, di luar dokumen upstream.

### GGUF Loader

Dockerfile di repo ini mendukung instalasi node pack `ComfyUI-GGUF` saat build.

- Build arg: `INSTALL_GGUF_NODES=true|false` (default: `true`)

### `MODEL_TYPE=jawi-vision` + `CIVITAI_*`

Stage `downloader` di Dockerfile mendukung `MODEL_TYPE=jawi-vision` untuk mendownload model tertentu saat build.

Untuk download CivitAI (opsional):

- `CIVITAI_DOWNLOAD_URL_1` + `CIVITAI_FILENAME_1`
- `CIVITAI_DOWNLOAD_URL_2` + `CIVITAI_FILENAME_2`

Jika URL kosong, download akan dilewati.
