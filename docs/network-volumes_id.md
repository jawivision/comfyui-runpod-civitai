# Network Volumes & Path Model

Dokumen ini menjelaskan cara menggunakan **Network Volumes** RunPod dengan `worker-comfyui`, bagaimana path model di-resolve di dalam container, dan cara debugging saat model tidak terdeteksi.

> **Cakupan**
>
> Instruksi ini berlaku untuk **serverless endpoint** yang menggunakan worker ini. Pod me-mount network volume pada `/workspace` secara default, sedangkan serverless worker melihatnya pada `/runpod-volume`.

## Mapping Direktori

Untuk **serverless endpoint**:

- Root network volume di-mount di: `/runpod-volume`
- Model ComfyUI diharapkan berada di: `/runpod-volume/models/...`

Untuk **Pods**:

- Root network volume di-mount di: `/workspace`
- Path model ComfyUI yang ekuivalen: `/workspace/models/...`

Jika Anda menggunakan S3-compatible API, mapping path yang sama adalah:

- Serverless: `/runpod-volume/my-folder/file.txt`
- Pod: `/workspace/my-folder/file.txt`
- S3 API: `s3://<NETWORK_VOLUME_ID>/my-folder/file.txt`

## Struktur Direktori yang Diharapkan

Model harus diletakkan dalam struktur berikut pada network volume Anda:

```text
/runpod-volume/
└── models/
    ├── checkpoints/      # Checkpoint Stable Diffusion (.safetensors, .ckpt)
    ├── loras/            # File LoRA (.safetensors, .pt)
    ├── vae/              # Model VAE (.safetensors, .pt)
    ├── clip/             # Model CLIP (.safetensors, .pt)
    ├── clip_vision/      # Model CLIP Vision
    ├── controlnet/       # Model ControlNet (.safetensors, .pt)
    ├── embeddings/       # Embedding textual inversion (.safetensors, .pt)
    ├── upscale_models/   # Model upscaling (.safetensors, .pt)
    ├── unet/             # Model UNet
    └── configs/          # Config model (.yaml, .json)
```

> **Catatan**
>
> Buat subdirektori hanya yang Anda perlukan; folder kosong atau tidak ada tetap tidak masalah.

## Ekstensi File yang Didukung

ComfyUI hanya mengenali file dengan ekstensi tertentu ketika melakukan scan direktori model.

| Jenis Model       | Ekstensi yang Didukung                             |
| ---------------- | --------------------------------------------------- |
| Checkpoints      | `.safetensors`, `.ckpt`, `.pt`, `.pth`, `.bin`      |
| LoRAs            | `.safetensors`, `.pt`                               |
| VAE              | `.safetensors`, `.pt`, `.bin`                       |
| CLIP             | `.safetensors`, `.pt`, `.bin`                       |
| ControlNet       | `.safetensors`, `.pt`, `.pth`, `.bin`               |
| Embeddings       | `.safetensors`, `.pt`, `.bin`                       |
| Upscale Models   | `.safetensors`, `.pt`, `.pth`                       |

File dengan ekstensi lain (misalnya `.txt`, `.zip`) akan **diabaikan** oleh discovery model ComfyUI.

## Masalah Umum

- **Direktori root salah**
  - Model diletakkan langsung di `/runpod-volume/checkpoints/...` alih-alih `/runpod-volume/models/checkpoints/...`.
- **Ekstensi tidak benar**
  - File yang dinamai tanpa salah satu ekstensi yang didukung akan dilewati.
- **Direktori kosong**
  - Tidak ada file model pada `models/checkpoints` (atau folder lainnya).
- **Volume tidak terpasang**
  - Endpoint dibuat tanpa memilih network volume pada **Advanced → Select Network Volume**.

Jika salah satu kondisi di atas terjadi, ComfyUI akan gagal menemukan model dari network volume tanpa menampilkan error.

## Debugging dengan `NETWORK_VOLUME_DEBUG`

Worker menyediakan mode debug opsional yang dikendalikan lewat variabel lingkungan `NETWORK_VOLUME_DEBUG`.

### Kapan Digunakan

Aktifkan ini ketika:

- Model pada network volume tidak muncul di ComfyUI
- Anda curiga struktur direktori atau ekstensi file salah
- Anda ingin cepat memverifikasi apa yang benar-benar terlihat oleh worker pada `/runpod-volume`

### Cara Mengaktifkan

1. Buka serverless **Endpoint → Manage → Edit**.
2. Pada **Environment Variables**, tambahkan:

   - `NETWORK_VOLUME_DEBUG=true`

3. Simpan dan tunggu worker restart (atau scale ke nol lalu naikkan lagi).
4. Kirim request apa pun ke endpoint (bahkan minimal) untuk memicu diagnostik.

### Membaca Diagnostik

Saat aktif, setiap request akan mencetak laporan detail ke log worker, contohnya:

```text
======================================================================
NETWORK VOLUME DIAGNOSTICS (NETWORK_VOLUME_DEBUG=true)
======================================================================

[1] Checking extra_model_paths.yaml configuration...
    ✓ FOUND: /comfyui/extra_model_paths.yaml

[2] Checking network volume mount at /runpod-volume...
    ✓ MOUNTED: /runpod-volume

[3] Checking directory structure...
    ✓ FOUND: /runpod-volume/models

[4] Scanning model directories...

    checkpoints/:
      - my-model.safetensors (6.5 GB)

    loras/:
      - style-lora.safetensors (144.2 MB)

[5] Summary
    ✓ Models found on network volume!
======================================================================
```

Jika ada masalah, diagnostik akan menyoroti hal tersebut, misalnya:

- Direktori `models/` tidak ada
- Tidak ada file model valid di subdirektori mana pun
- File ada tetapi diabaikan karena ekstensi salah

### Menonaktifkan Debug Mode

Setelah masalah selesai, nonaktifkan diagnostik agar log tetap bersih:

- Hapus variabel lingkungan `NETWORK_VOLUME_DEBUG`, **atau**
- Set `NETWORK_VOLUME_DEBUG=false`

Ini mengembalikan worker ke perilaku normal tanpa menambah noise di log.

