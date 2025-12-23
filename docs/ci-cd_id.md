# CI/CD

Project ini menyertakan workflow GitHub Actions untuk otomatis membuild dan mendeploy image Docker ke Docker Hub.

## Deployment Otomatis ke Docker Hub dengan GitHub Actions

Repository ini memiliki dua workflow yang berada di direktori `.github/workflows`:

- [`dev.yml`](../.github/workflows/dev.yml): Membuat image (base, sdxl, sd3, varian flux) dan push ke Docker Hub dengan tag `<image_name>:dev` pada setiap push ke branch `main`.
- [`release.yml`](../.github/workflows/release.yml): Membuat image dan push ke Docker Hub dengan tag `<image_name>:latest` dan `<image_name>:<release_version>` (misalnya `worker-comfyui:3.7.0`). Workflow ini hanya terpicu ketika ada release baru dibuat di GitHub.

### Konfigurasi untuk Fork Anda

Jika Anda mem-fork repository ini dan ingin memakai action tersebut untuk publish image ke akun Docker Hub Anda sendiri, Anda perlu mengonfigurasi hal berikut pada pengaturan repository GitHub Anda:

1.  **Secrets** (`Settings > Secrets and variables > Actions > New repository secret`):

    | Nama Secret               | Deskripsi                                                                 | Contoh Nilai         |
    | ------------------------- | ------------------------------------------------------------------------- | -------------------- |
    | `DOCKERHUB_USERNAME`      | Username Docker Hub Anda.                                                 | `your-dockerhub-id`  |
    | `DOCKERHUB_TOKEN`         | Access token Docker Hub Anda dengan izin read/write.                      | `dckr_pat_...`       |
    | `HUGGINGFACE_ACCESS_TOKEN`| Token READ dari Hugging Face (dibutuhkan hanya untuk build SD3).          | `hf_...`             |

2.  **Variables** (`Settings > Secrets and variables > Actions > New repository variable`):

    | Nama Variabel   | Deskripsi                                                                   | Contoh Nilai                 |
    | --------------- | --------------------------------------------------------------------------- | ---------------------------- |
    | `DOCKERHUB_REPO`| Repository (namespace) target di Docker Hub tempat image akan dipush.       | `your-dockerhub-id`          |
    | `DOCKERHUB_IMG` | Nama dasar image yang akan dipush ke Docker Hub.                            | `my-custom-worker-comfyui`  |

Jika secrets dan variables tersebut dikonfigurasi, action akan mem-push image yang dibuild (misalnya `your-dockerhub-id/my-custom-worker-comfyui:dev`, `your-dockerhub-id/my-custom-worker-comfyui:1.0.0`, `your-dockerhub-id/my-custom-worker-comfyui:latest`) ke akun Docker Hub Anda saat terpicu.

