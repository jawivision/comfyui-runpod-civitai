# Development dan Pengujian Lokal

Panduan ini membahas penyiapan environment lokal untuk development dan pengujian `worker-comfyui`.

Kedua jenis tes akan menggunakan data dari [`test_input.json`](../test_input.json), jadi lakukan perubahan di sana untuk menguji input workflow yang berbeda dengan benar.

## Setup

### Prasyarat

1.  Python >= 3.10
2.  `pip` (installer package Python)
3.  Tool virtual environment (seperti `venv`)

### Langkah

1.  **Clone repository** (jika belum):
    ```bash
    git clone https://github.com/runpod-workers/worker-comfyui.git
    cd worker-comfyui
    ```
2.  **Buat virtual environment**:
    ```bash
    python -m venv .venv
    ```
3.  **Aktifkan virtual environment**:
    - **Windows (Command Prompt/PowerShell)**:
      ```bash
      .\.venv\Scripts\activate
      ```
    - **macOS / Linux (Bash/Zsh)**:
      ```bash
      source ./.venv/bin/activate
      ```
4.  **Install dependensi**:
    ```bash
    pip install -r requirements.txt
    ```

### Setup untuk Windows (menggunakan WSL2)

Menjalankan Docker dengan akselerasi GPU di Windows biasanya membutuhkan WSL2 (Windows Subsystem for Linux).

1.  **Install WSL2 dan distro Linux** (misalnya Ubuntu) mengikuti [panduan resmi Microsoft](https://learn.microsoft.com/en-us/windows/wsl/install). Umumnya Anda tidak membutuhkan GUI support untuk ini.
2.  **Buka terminal distro Linux Anda** (misalnya buka Ubuntu dari Start menu atau ketik `wsl` di Command Prompt/PowerShell).
3.  **Update package** di dalam WSL:
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
4.  **Install Docker Engine di WSL**:
    - Ikuti [panduan instalasi Docker resmi](https://docs.docker.com/engine/install/#server) untuk distro Linux Anda (misalnya Ubuntu).
    - **Penting:** Tambahkan user Anda ke grup `docker` agar tidak perlu `sudo` untuk setiap perintah Docker: `sudo usermod -aG docker $USER`. Anda mungkin perlu menutup dan membuka ulang terminal agar perubahan berlaku.
5.  **Install Docker Compose** (jika tidak termasuk dalam Docker Engine):
    ```bash
    sudo apt-get update
    sudo apt-get install docker-compose-plugin # Atau gunakan metode binary standalone jika diinginkan
    ```
6.  **Install NVIDIA Container Toolkit di WSL**:
    - Ikuti [panduan instalasi NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html), pastikan memilih langkah yang sesuai untuk distro Linux Anda di dalam WSL.
    - Konfigurasikan Docker agar menggunakan NVIDIA runtime sebagai default jika diinginkan, atau tentukan runtime saat menjalankan container.
7.  **Aktifkan akselerasi GPU di WSL**:
    - Pastikan Anda sudah menginstall driver NVIDIA terbaru di Windows host.
    - Ikuti [panduan NVIDIA untuk CUDA di WSL](https://docs.nvidia.com/cuda/wsl-user-guide/index.html).

Setelah langkah-langkah di atas selesai, Anda seharusnya bisa menjalankan perintah Docker (termasuk `docker-compose`) dari terminal WSL dengan akses GPU.

> [!NOTE]
>
> - Umumnya disarankan menjalankan perintah Docker (`docker build`, `docker-compose up`) dari terminal WSL agar konsisten dengan environment container berbasis Linux.
> - Mengakses URL `localhost` (seperti local API atau ComfyUI) dari browser Windows saat service berjalan di WSL biasanya berfungsi, tetapi konfigurasi jaringan kadang bisa menimbulkan masalah.

## Menguji RunPod Handler

Unit test tersedia untuk memverifikasi logika utama di `handler.py`.

- **Jalankan semua tes**:
  ```bash
  python -m unittest discover tests/
  ```
- **Jalankan file tes tertentu**:
  ```bash
  python -m unittest tests.test_handler
  ```
- **Jalankan test case atau method tertentu**:

  ```bash
  # Contoh: jalankan semua tes di class TestRunpodWorkerComfy
  python -m unittest tests.test_handler.TestRunpodWorkerComfy

  # Contoh: jalankan satu test method
  python -m unittest tests.test_handler.TestRunpodWorkerComfy.test_s3_upload
  ```

## Simulasi API Lokal (menggunakan Docker Compose)

Untuk development lokal dan pengujian end-to-end yang lebih baik, Anda bisa menjalankan environment lokal menggunakan Docker Compose yang mencakup worker dan instance ComfyUI.

> [!IMPORTANT]
>
> - Saat ini ini membutuhkan **NVIDIA GPU** serta driver + NVIDIA Container Toolkit yang sudah terkonfigurasi dengan benar (lihat setup Windows di atas jika diperlukan).
> - Pastikan Docker sudah berjalan.

**Langkah:**

1.  **Set Variabel Lingkungan (Opsional namun Disarankan):**
    - Walaupun `docker-compose.yml` mengatur `SERVE_API_LOCALLY=true` secara default, Anda mungkin mengelola variabel lingkungan secara eksternal (misalnya lewat file `.env`).
    - Pastikan variabel lingkungan `SERVE_API_LOCALLY` di-set ke `true` untuk service `worker` jika Anda memodifikasi file compose atau menggunakan `.env`.
2.  **Jalankan service**:
    ```bash
    # Dari direktori root project
    docker-compose up --build
    ```
    - Flag `--build` memastikan image dibuild secara lokal menggunakan kondisi kode terbaru dan `Dockerfile`.
    - Ini akan menjalankan dua container: `comfyui` dan `worker`.

### Mengakses Worker API Lokal

- Dengan stack Docker Compose berjalan, API RunPod simulasi dari worker dapat diakses di: [http://localhost:8000](http://localhost:8000)
- Anda bisa mengirim POST request ke `http://localhost:8000/run` atau `http://localhost:8000/runsync` dengan payload JSON yang sama seperti yang diharapkan oleh endpoint RunPod.
- Membuka [http://localhost:8000/docs](http://localhost:8000/docs) di browser akan menampilkan dokumentasi FastAPI yang dibuat otomatis (Swagger UI), sehingga Anda bisa mencoba API secara langsung.

### Mengakses ComfyUI Lokal

- Instance ComfyUI di container `comfyui` bisa diakses langsung di: [http://localhost:8188](http://localhost:8188)
- Ini berguna untuk debugging workflow atau mengamati state ComfyUI saat menguji worker.

### Menghentikan Environment Lokal

- Tekan `Ctrl+C` di terminal tempat `docker-compose up` berjalan.
- Untuk memastikan container dihapus, Anda bisa menjalankan: `docker-compose down`

