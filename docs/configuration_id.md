# Konfigurasi

Dokumen ini menjelaskan variabel lingkungan yang tersedia untuk mengonfigurasi `worker-comfyui`.

## Konfigurasi Umum

| Variabel Lingkungan  | Deskripsi                                                                                                                                                                                                                   | Default |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `REFRESH_WORKER`     | Jika `true`, worker pod akan berhenti setelah setiap job selesai untuk memastikan state bersih untuk job berikutnya. Lihat [dokumentasi RunPod](https://docs.runpod.io/docs/handler-additional-controls#refresh-worker) untuk detail. | `false` |
| `SERVE_API_LOCALLY`  | Jika `true`, mengaktifkan server HTTP lokal yang mensimulasikan environment RunPod untuk development dan pengujian. Lihat [Panduan Development](development.md#local-api) untuk detail lebih lanjut.                       | `false` |
| `COMFY_ORG_API_KEY`  | API key Comfy.org untuk mengaktifkan ComfyUI API Nodes. Jika di-set, key ini dikirim bersama setiap workflow; client bisa override per request melalui `input.api_key_comfy_org`.                                           | –       |

## Konfigurasi Logging

| Variabel Lingkungan     | Deskripsi                                                                                                                                                      | Default |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `COMFY_LOG_LEVEL`       | Mengatur tingkat verbosity logging internal ComfyUI. Opsi: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. Gunakan `DEBUG` untuk troubleshooting, `INFO` untuk production. | `DEBUG` |
| `NETWORK_VOLUME_DEBUG`  | Mengaktifkan diagnostik network volume yang detail di log worker. Berguna untuk debugging masalah path model. Lihat [Network Volumes & Model Paths](network-volumes.md). | `false` |

## Konfigurasi Debugging

| Variabel Lingkungan              | Deskripsi                                                                                                             | Default |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- | ------- |
| `WEBSOCKET_RECONNECT_ATTEMPTS`  | Jumlah percobaan reconnect websocket ketika koneksi terputus saat eksekusi job.                                      | `5`     |
| `WEBSOCKET_RECONNECT_DELAY_S`   | Delay (detik) antar percobaan reconnect websocket.                                                                    | `3`     |
| `WEBSOCKET_TRACE`               | Mengaktifkan tracing frame websocket level rendah untuk debugging protokol. Set `true` hanya saat mendiagnosis masalah koneksi. | `false` |

## Konfigurasi Upload AWS S3

Konfigurasikan variabel-variabel ini **hanya** jika Anda ingin worker mengupload gambar hasil secara langsung ke bucket AWS S3. Jika variabel-variabel ini tidak di-set, gambar akan dikembalikan sebagai string base64 pada response API.

- **Prasyarat:**
  - Bucket AWS S3 di region yang Anda inginkan.
  - AWS IAM user dengan programmatic access (Access Key ID dan Secret Access Key).
  - Permission pada IAM user yang mengizinkan `s3:PutObject` (dan kemungkinan `s3:PutObjectAcl` jika Anda butuh ACL tertentu) pada bucket tujuan.

| Variabel Lingkungan        | Deskripsi                                                                                                                              | Contoh                                                      |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `BUCKET_ENDPOINT_URL`      | URL endpoint lengkap bucket S3 Anda. **Harus di-set untuk mengaktifkan upload S3.**                                                    | `https://<nama-bucket-anda>.s3.<aws-region>.amazonaws.com` |
| `BUCKET_ACCESS_KEY_ID`     | AWS access key ID milik IAM user yang punya izin tulis ke bucket. Wajib jika `BUCKET_ENDPOINT_URL` di-set.                             | `AKIAIOSFODNN7EXAMPLE`                                     |
| `BUCKET_SECRET_ACCESS_KEY` | AWS secret access key milik IAM user. Wajib jika `BUCKET_ENDPOINT_URL` di-set.                                                         | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`                 |

**Catatan:** Upload menggunakan helper library Python `runpod` yaitu `rp_upload.upload_image`, yang menangani pembuatan path unik dalam bucket berdasarkan `job_id`.

### Contoh Response S3

Jika variabel lingkungan S3 (`BUCKET_ENDPOINT_URL`, `BUCKET_ACCESS_KEY_ID`, `BUCKET_SECRET_ACCESS_KEY`) sudah dikonfigurasi dengan benar, response sukses job akan terlihat seperti ini:

```json
{
  "id": "sync-uuid-string",
  "status": "COMPLETED",
  "output": {
    "images": [
      {
        "filename": "ComfyUI_00001_.png",
        "type": "s3_url",
        "data": "https://your-bucket-name.s3.your-region.amazonaws.com/sync-uuid-string/ComfyUI_00001_.png"
      }
      // Gambar tambahan yang dihasilkan workflow akan muncul di sini
    ]
    // Key "errors" bisa muncul di sini jika ada isu non-fatal
  },
  "delayTime": 123,
  "executionTime": 4567
}
```

Field `data` berisi URL presigned ke file gambar yang diupload pada bucket S3 Anda. Path biasanya mencakup job ID.

