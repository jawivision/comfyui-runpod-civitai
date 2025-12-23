# Kustomisasi

Panduan ini membahas metode untuk menambahkan model Anda sendiri, custom node, dan file input statis ke dalam `worker-comfyui` kustom.

> [!TIP]
>
> **Mencari cara termudah untuk deploy workflow kustom?**
>
> [ComfyUI-to-API](https://comfy.getrunpod.io) secara otomatis menghasilkan Dockerfile kustom dan repository GitHub dari workflow ComfyUI Anda, sehingga menghilangkan setup manual seperti yang dijelaskan di bawah. Lihat [Dokumentasi ComfyUI-to-API](https://docs.runpod.io/community-solutions/comfyui-to-api/overview) untuk detail.
>
> Gunakan metode manual di bawah hanya jika Anda butuh kontrol yang lebih detail atau lebih suka mengelola semuanya sendiri.

---

Ada dua metode utama untuk kustomisasi **manual**:

1.  **Dockerfile Kustom (direkomendasikan untuk setup manual):** Buat `Dockerfile` Anda sendiri yang dimulai dengan `FROM` salah satu base image resmi `worker-comfyui`. Ini memungkinkan Anda “membake” custom node, model, dan file input tertentu langsung ke image menggunakan perintah `comfy-cli`. **Metode ini tidak mengharuskan Anda fork repository `worker-comfyui`.**
2.  **Network Volume:** Simpan model di network volume persisten yang dipasang ke endpoint RunPod Anda. Ini berguna jika Anda sering mengganti model atau punya model yang sangat besar dan tidak ingin dimasukkan ke proses build image.

## Metode 1: Dockerfile Kustom

> [!NOTE]
>
> Metode ini TIDAK mengharuskan Anda fork repository `worker-comfyui`.

Ini adalah pendekatan yang paling fleksibel dan direkomendasikan untuk membuat environment worker kustom yang reproducible.

1.  **Buat `Dockerfile`:** Di direktori project Anda sendiri, buat file bernama `Dockerfile`.
2.  **Mulai dari Base Image:** Awali `Dockerfile` Anda dengan merujuk salah satu base image resmi. Disarankan memakai tag `-base` karena memberikan instalasi ComfyUI yang bersih beserta tool yang dibutuhkan seperti `comfy-cli`, namun tanpa model yang sudah dipaketkan.
    ```Dockerfile
    # mulai dari base image yang bersih (ganti <version> dengan [release](https://github.com/runpod-workers/worker-comfyui/releases) yang diinginkan)
    FROM runpod/worker-comfyui:<version>-base
    ```
3.  **Install Custom Node:** Gunakan perintah `comfy-node-install` (kami memperkenalkan tool CLI sendiri di sini, karena ada [masalah dengan comfy-cli yang tidak menampilkan error saat instalasi](https://github.com/Comfy-Org/comfy-cli/pull/275)) untuk menambahkan custom node berdasarkan nama atau URL, lihat [Comfy Registry](https://registry.comfy.org) untuk menemukan nama yang benar. Anda bisa menuliskan beberapa node.
    ```Dockerfile
    # install custom nodes menggunakan comfy-cli
    RUN comfy-node-install comfyui-kjnodes comfyui-ic-light
    ```
4.  **Download Model:** Gunakan perintah `comfy model download` untuk mengunduh model dan menaruhnya di direktori ComfyUI yang benar.

    ```Dockerfile
    # download model menggunakan comfy-cli
    RUN comfy model download --url https://huggingface.co/KamCastle/jugg/resolve/main/juggernaut_reborn.safetensors --relative-path models/checkpoints --filename juggernaut_reborn.safetensors
    ```

> [!NOTE]
>
> Pastikan Anda menggunakan `--relative-path` yang sesuai dengan struktur direktori model ComfyUI (dimulai dengan `models/<folder>`):
>
> checkpoints, clip, clip_vision, configs, controlnet, diffusers, embeddings, gligen, hypernetworks, loras, style_models, unet, upscale_models, vae, vae_approx, animatediff_models, animatediff_motion_lora, ipadapter, photomaker, sams, insightface, facerestore_models, facedetection, mmdets, instantid

5.  **Tambahkan File Input Statis (Opsional):** Jika workflow Anda selalu membutuhkan input image/mask/video tertentu, Anda dapat menyalinnya langsung ke image.

- Buat direktori `input/` pada folder yang sama dengan `Dockerfile`.
- Letakkan file statis Anda di dalam direktori `input/`.
- Tambahkan perintah `COPY` pada `Dockerfile`:

  ```Dockerfile
  # Salin file input statis lokal ke direktori input ComfyUI
  COPY input/ /comfyui/input/
  ```

- File-file ini kemudian bisa direferensikan di workflow Anda menggunakan node "Load Image" (atau sejenisnya) yang menunjuk ke nama file (misalnya `my_static_image.png`).

Setelah Anda membuat `Dockerfile` kustom, lihat [Panduan Deployment](deployment.md#deploying-custom-setups) untuk instruksi build, push, dan deploy image kustom Anda ke RunPod.

### Contoh `Dockerfile` Kustom Lengkap

```Dockerfile
# mulai dari base image yang bersih (ganti <version> dengan release yang diinginkan)
FROM runpod/worker-comfyui:5.1.0-base

# install custom nodes menggunakan comfy-cli
RUN comfy-node-install comfyui-kjnodes comfyui-ic-light comfyui_ipadapter_plus comfyui_essentials ComfyUI-Hangover-Nodes

# download model menggunakan comfy-cli
# "--filename" adalah yang Anda pakai di workflow ComfyUI
RUN comfy model download --url https://huggingface.co/KamCastle/jugg/resolve/main/juggernaut_reborn.safetensors --relative-path models/checkpoints --filename juggernaut_reborn.safetensors
RUN comfy model download --url https://huggingface.co/h94/IP-Adapter/resolve/main/models/ip-adapter-plus_sd15.bin --relative-path models/ipadapter --filename ip-adapter-plus_sd15.bin
RUN comfy model download --url https://huggingface.co/shiertier/clip_vision/resolve/main/SD15/model.safetensors --relative-path models/clip_vision --filename models.safetensors
RUN comfy model download --url https://huggingface.co/lllyasviel/ic-light/resolve/main/iclight_sd15_fcon.safetensors --relative-path models/diffusion_models --filename iclight_sd15_fcon.safetensors

# Salin file input statis lokal ke direktori input ComfyUI (hapus jika tidak diperlukan)
# Diasumsikan Anda punya folder 'input' di sebelah Dockerfile
COPY input/ /comfyui/input/
```

## Metode 2: Network Volume

Menggunakan Network Volume terutama berguna jika Anda ingin mengelola **model** terpisah dari image worker, khususnya jika ukurannya besar atau sering berubah.

1.  **Buat Network Volume**:
    - Ikuti [panduan RunPod Network Volumes](https://docs.runpod.io/pods/storage/create-network-volumes) untuk membuat volume di region yang sama dengan endpoint Anda.
2.  **Isi Volume dengan Model**:
    - Gunakan salah satu metode di panduan RunPod (misalnya Pod sementara + `wget`, upload langsung, atau S3-compatible API) untuk menaruh file model Anda ke struktur direktori ComfyUI yang benar **di dalam volume**.
    - Untuk **serverless endpoint**, network volume dipasang pada `/runpod-volume`, dan ComfyUI mengharapkan model berada di bawah `/runpod-volume/models/...`. Lihat [Network Volumes & Model Paths](network-volumes.md) untuk struktur yang tepat dan tips debugging.
      ```bash
      # Contoh struktur di dalam Network Volume (tampilan pada serverless worker):
      # /runpod-volume/models/checkpoints/your_model.safetensors
      # /runpod-volume/models/loras/your_lora.pt
      # /runpod-volume/models/vae/your_vae.safetensors
      ```
    - **Penting:** Pastikan model berada di subdirektori yang benar (misalnya checkpoint di `models/checkpoints`, LoRA di `models/loras`). Jika model tidak terdeteksi, aktifkan `NETWORK_VOLUME_DEBUG` seperti dijelaskan di [Network Volumes & Model Paths](network-volumes.md).
3.  **Konfigurasikan Endpoint Anda**:
    - Gunakan Network Volume pada konfigurasi endpoint:
      - Buat endpoint baru atau update endpoint yang sudah ada (lihat [Panduan Deployment](deployment.md)).
      - Pada konfigurasi endpoint, di `Advanced > Select Network Volume`, pilih Network Volume Anda.

> [!NOTE]
>
> - Ketika Network Volume terpasang dengan benar, ComfyUI yang berjalan di dalam container worker akan otomatis mendeteksi dan memuat model dari direktori standar (`/runpod-volume/models/...`) pada volume tersebut (untuk serverless worker). Untuk detail mapping direktori dan troubleshooting, lihat [Network Volumes & Model Paths](network-volumes.md).
> - Metode ini **tidak cocok untuk instalasi custom node**; gunakan metode Dockerfile Kustom untuk itu.

