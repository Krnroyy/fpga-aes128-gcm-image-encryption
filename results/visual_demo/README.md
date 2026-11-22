# FPGA AES-128-GCM Image Encryption — Visual Evidence

This directory contains output artifacts from the protected-chunk FPGA image encryption demonstration on the AMD/Xilinx ZCU104.

## Demonstration runs

| Image size | Input image | Encrypted visualization | Recovered image |
|---|---|---|---|
| 17×19 | input.png | encrypted.png | recovered.png |
| 256×256 | input.png | encrypted.png | recovered.png |
| 512×512 | input.png | encrypted.png | recovered.png |

Each image-size directory contains:

- `input.png` — original input image.
- `encrypted.png` — visualization of the encrypted output.
- `ciphertext.bin` — raw encrypted ciphertext bytes.
- `recovered.png` — reconstructed image after successful decryption.
- `recovered.bin` — raw recovered plaintext bytes.
- `protected_chunk_summary.json` — run summary and recorded results.
- `uart_transcript.txt` — UART log from the corresponding demonstration.

## Important distinction

`encrypted.png` is an image visualization of encrypted data. The actual encrypted byte stream is `ciphertext.bin`.

The UART transcripts and JSON summaries provide supporting execution evidence. Consult the project verification documentation for the detailed authentication, tamper-rejection, and zeroization test results.

## Reproducibility

The files are preserved from the corresponding local demonstration runs. Their source and copied versions were checked using SHA-256 during the copy operation.

No AES secret key is included in this directory.
