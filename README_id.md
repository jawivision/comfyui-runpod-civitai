# worker-comfyui

> [ComfyUI](https://github.com/comfyanonymous/ComfyUI) sebagai serverless API di [RunPod](https://www.runpod.io/)

<p align="center">
  <img src="assets/worker_sitting_in_comfy_chair.jpg" title="Worker sitting in comfy chair" />
</p>

[![RunPod](https://api.runpod.io/badge/runpod-workers/worker-comfyui)](https://www.runpod.io/console/hub/runpod-workers/worker-comfyui)

---

Project ini memungkinkan Anda menjalankan workflow ComfyUI sebagai endpoint serverless API di platform RunPod. Kirim workflow via API dan terima hasil gambar sebagai string base64 atau URL S3 (jika Anda mengonfigurasi upload S3).

## Daftar Isi

- [Quickstart](#quickstart)
- [Docker Image yang Tersedia](#docker-image-yang-tersedia)
- [Spesifikasi API](#spesifikasi-api)
- [Cara Pakai](#cara-pakai)
- [Cara Mendapatkan JSON Workflow](#cara-mendapatkan-json-workflow)
- [Dokumentasi Lanjutan](#dokumentasi-lanjutan)
- [Tambahan Kustomisasi (Repo Ini)](#tambahan-kustomisasi-repo-ini)

---

## Quickstart

1. 🐳 Pilih salah satu [Docker image yang tersedia](#docker-image-yang-tersedia) untuk serverless endpoint Anda (misalnya `runpod/worker-comfyui:<version>-sd3`).
2. 📄 Ikuti panduan [Deployment](docs/deployment.md) (atau versi Indonesia: `docs/deployment_id.md`) untuk membuat template dan endpoint di RunPod.
3. ⚙️ (Opsional) Konfigurasi worker (mis. untuk upload S3) lewat environment variables. Lihat [Configuration Guide](docs/configuration.md).
4. 🧪 Pilih contoh workflow dari `test_resources/workflows/` atau gunakan workflow Anda sendiri (lihat [Cara Mendapatkan JSON Workflow](#cara-mendapatkan-json-workflow)).
5. 🚀 Ikuti langkah [Cara Pakai](#cara-pakai) untuk memanggil endpoint yang sudah dideploy.

## Docker Image yang Tersedia

Image-image ini tersedia di Docker Hub pada `runpod/worker-comfyui`:

- **`runpod/worker-comfyui:<version>-base`**: instalasi ComfyUI bersih tanpa model bawaan.
- **`runpod/worker-comfyui:<version>-flux1-schnell`**: berisi checkpoint, text encoders, dan VAE untuk [FLUX.1 schnell](https://huggingface.co/black-forest-labs/FLUX.1-schnell).
- **`runpod/worker-comfyui:<version>-flux1-dev`**: berisi checkpoint, text encoders, dan VAE untuk [FLUX.1 dev](https://huggingface.co/black-forest-labs/FLUX.1-dev).
- **`runpod/worker-comfyui:<version>-sdxl`**: berisi checkpoint dan VAE untuk [Stable Diffusion XL](https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0).
- **`runpod/worker-comfyui:<version>-sd3`**: berisi checkpoint untuk [Stable Diffusion 3 medium](https://huggingface.co/stabilityai/stable-diffusion-3-medium).

Ganti `<version>` dengan tag release terbaru (lihat halaman releases repo upstream).

## Spesifikasi API

Worker ini mengekspos endpoint standar RunPod serverless (`/run`, `/runsync`, `/health`). Secara default hasil gambar dikembalikan sebagai base64. Anda dapat mengonfigurasi agar gambar diupload ke S3 dengan mengatur environment variables tertentu (lihat `docs/configuration.md`).

Gunakan `/runsync` untuk request sinkron (menunggu job selesai dan hasil dikembalikan langsung). Gunakan `/run` untuk request asinkron (langsung mengembalikan job id; Anda perlu polling `/status` untuk mengambil hasilnya).

### Input

```json
{
  "input": {
    "workflow": {
      "6": {
        "inputs": {
          "text": "a ball on the table",
          "clip": ["30", 1]
        },
        "class_type": "CLIPTextEncode",
        "_meta": {
          "title": "CLIP Text Encode (Positive Prompt)"
        }
      }
    },
    "images": [
      {
        "name": "input_image_1.png",
        "image": "data:image/png;base64,iVBOR..."
      }
    ]
  }
}
```

Tabel berikut menjelaskan field di dalam object `input`:

| Path Field                | Tipe   | Wajib | Deskripsi                                                                                                                                     |
| ------------------------- | ------ | ----- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `input`                   | Object | Ya    | Object teratas yang berisi data request.                                                                                                     |
| `input.workflow`          | Object | Ya    | Workflow ComfyUI yang diekspor dalam [format yang dibutuhkan](#cara-mendapatkan-json-workflow).                                               |
| `input.images`            | Array  | Tidak | Array opsional gambar input. Setiap gambar diupload ke folder `input` ComfyUI dan dapat direferensikan dengan `name` pada workflow.          |
| `input.comfy_org_api_key` | String | Tidak | API key Comfy.org opsional untuk API Nodes. Meng-override env var `COMFY_ORG_API_KEY` bila keduanya diisi.                                    |

#### Object `input.images`

Setiap item dalam `input.images` harus memiliki:

| Nama Field | Tipe   | Wajib | Deskripsi                                                                                                                     |
| ---------- | ------ | ----- | ---------------------------------------------------------------------------------------------------------------------------- |
| `name`     | String | Ya    | Nama file untuk direferensikan dalam workflow (mis. node “Load Image”). Harus unik di dalam array.                            |
| `image`    | String | Ya    | String base64 gambar. Prefix data URI (mis. `data:image/png;base64,`) opsional dan akan ditangani secara benar.               |

> **Catatan batas ukuran**
>
> RunPod memiliki batas ukuran request (mis. 10MB untuk `/run`, 20MB untuk `/runsync`). Input gambar base64 yang besar dapat melebihi batas ini. Lihat dokumentasi RunPod terkait limit request.

### Output

> **Perubahan format output (5.0.0+)**
>
> Versi `< 5.0.0` mengembalikan data gambar utama (URL S3 atau base64) langsung dalam `output.message`.
> Mulai `5.0.0`, format output berubah seperti contoh berikut.

```json
{
  "id": "sync-uuid-string",
  "status": "COMPLETED",
  "output": {
    "images": [
      {
        "filename": "ComfyUI_00001_.png",
        "type": "base64",
        "data": "iVBORw0KGgoAAAANSUhEUg..."
      }
    ]
  },
  "delayTime": 123,
  "executionTime": 4567
}
```

| Path Field      | Tipe             | Wajib | Deskripsi                                                                                                        |
| --------------- | ---------------- | ----- | --------------------------------------------------------------------------------------------------------------- |
| `output`        | Object           | Ya    | Object berisi hasil eksekusi job.                                                                               |
| `output.images` | Array of Object  | Tidak | Ada jika workflow menghasilkan gambar. Berisi daftar output image.                                               |
| `output.errors` | Array of String  | Tidak | Ada jika terdapat error/warning non-fatal (mis. upload S3 gagal, data hilang).                                   |

#### `output.images`

Setiap item di `output.images` memiliki struktur:

| Nama Field  | Tipe   | Deskripsi                                                                                           |
| ----------  | ------ | -------------------------------------------------------------------------------------------------- |
| `filename`  | String | Nama file yang diberikan ComfyUI saat generasi.                                                    |
| `type`      | String | Format data: `"base64"` atau `"s3_url"` (bila upload S3 dikonfigurasi).                            |
| `data`      | String | Berisi string base64 atau URL S3 dari file yang diupload.                                          |

> `output.images` berisi daftar semua gambar output (di luar gambar temporer). Klien perlu menangani bentuk list ini.

## Cara Pakai

Untuk berinteraksi dengan endpoint RunPod yang sudah dideploy:

1. **Dapatkan API Key:** buat API key di RunPod User Settings (bagian `API Keys`).
2. **Dapatkan Endpoint ID:** lihat endpoint id pada halaman Serverless Endpoints atau halaman Overview endpoint Anda.

### Generate Gambar (Contoh Sync)

Kirim workflow ke endpoint `/runsync` (menunggu sampai selesai). Ganti `<api_key>` dan `<endpoint_id>`. Nilai `-d` harus berisi JSON input sesuai format [Input](#input).

```bash
curl -X POST \
  -H "Authorization: Bearer <api_key>" \
  -H "Content-Type: application/json" \
  -d '{"input":{"workflow":{... workflow JSON Anda ...}}}' \
  https://api.runpod.ai/v2/<endpoint_id>/runsync
```

Anda juga dapat memakai `/run` untuk job asinkron lalu polling `/status`, atau menambahkan `webhook` di request agar mendapat notifikasi saat job selesai.

Lihat `test_input.json` untuk contoh input yang lengkap.

## Cara Mendapatkan JSON Workflow

Untuk mendapatkan JSON `workflow` yang benar untuk API:

1. Buka ComfyUI di browser.
2. Pada menu atas, pilih `Workflow > Export (API)`.
3. File `workflow.json` akan terunduh. Gunakan isi file tersebut sebagai nilai `input.workflow` saat request API.

## Dokumentasi Lanjutan

- **Deployment Guide:** `docs/deployment.md` (EN) dan `docs/deployment_id.md` (ID)
- **Configuration Guide:** `docs/configuration.md`
- **Customization Guide:** `docs/customization.md`
- **Development Guide:** `docs/development.md`
- **CI/CD Guide:** `docs/ci-cd.md`
- **Acknowledgments:** `docs/acknowledgments.md`

## Tambahan Kustomisasi (Repo Ini)

Bagian ini adalah tambahan khusus untuk repo Anda (di luar dokumen upstream) agar pembaca memahami perbedaan image custom yang sudah disesuaikan.

### GGUF Loader (ComfyUI-GGUF)

Dockerfile di repo ini dapat memasang node pack `ComfyUI-GGUF` (mis. `UnetLoaderGGUF`, `DualCLIPLoaderGGUF`) saat build image.

- Build arg: `INSTALL_GGUF_NODES=true|false` (default: `true`)

### `MODEL_TYPE=jawi-vision` + `CIVITAI_*` build args

Pada stage `downloader` di `Dockerfile`, Anda bisa memilih `MODEL_TYPE=jawi-vision` untuk mendownload model tambahan (CLIP/T5/VAE/LoRA) saat build image.

Untuk download dari CivitAI (opsional), gunakan build-arg berikut (URL boleh diisi satu atau dua; yang kosong akan dilewati):

- `CIVITAI_DOWNLOAD_URL_1` + `CIVITAI_FILENAME_1`
- `CIVITAI_DOWNLOAD_URL_2` + `CIVITAI_FILENAME_2`

Catatan:
- Download HF/CivitAI terjadi saat **build image**, bukan saat container runtime.
- Jangan commit token ke repo. Masukkan token via build-arg/secret pada pipeline build (mis. GitHub Actions), lalu hasil build dipush ke registry dan baru dideploy di RunPod.
