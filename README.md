# M22C Secure Suite

**M22C Secure Suite** is a high-security, zero-server, client-side web application designed for multi-layer file encryption and 3-stage recursive geometric fragmentation[cite: 1, 2]. Built entirely using modern web standards, all cryptographic processing occurs locally in the browser to ensure absolute confidentiality and zero server-side exposure[cite: 1, 2].

---

## Key Features

* **Zero-Server Architecture**: Performs all encryption and payload fragmentation locally inside the web browser[cite: 1, 2].
* **Multi-Layer AES-256-GCM Encryption**: Utilizes the native Web Crypto API with PBKDF2 key derivation (100,000 iterations, 16-byte random salt)[cite: 1, 2].
* **Geometric 4x4x4 Fragmentation**: Recursively splits encrypted payloads across 3 levels into exactly 64 binary blocks[cite: 1, 2].
* **Fisher-Yates Permutation**: Randomizes block storage sequence to obfuscate structural data relationships[cite: 1, 2].
* **`.m22c` Container Packaging**: Bundles shuffled binary fragments and an encrypted manifest (`manifest_tree.dat`) into a customized ZIP container[cite: 1, 2].
* **Streamlined UI/UX**: Features drag-and-drop file inputs, a 3-step workflow tracker, and color-coded feedback indicators (green for success, red for incomplete/error)[cite: 2].

---

## Architecture & Security Pipeline

The M22C encryption process follows a 3-stage processing flow:

```
               [ Input File ]
                     │
                     ▼
        [ Header Injection: "M22X" ]
                     │
                     ▼
  [ Stage 1: AES-256-GCM (Password 1) ]
                     │
                     ▼
  [ Geometric 4x4x4 Splitting (64 Blocks) ]
                     │
                     ▼
[ Stage 2 & 3: Multi-Pass Encryption (Pass 2 & Pass 3) ]
                     │
                     ▼
   [ Fisher-Yates Permutation & Manifest ]
                     │
                     ▼
     [ Packaging into .m22c Container ]
```

### 1. Header Injection & Primary Encryption
1. An `M22X` identifier header is injected into the raw file byte array[cite: 1].
2. Key 1 is derived from **Password 1** via PBKDF2-HMAC-SHA256[cite: 1, 2].
3. The prepended payload is encrypted using `AES-256-GCM`[cite: 1, 2].

### 2. Geometric Fragmentation & Multi-Pass Encryption
1. The encrypted payload is split into 4 initial chunks.
2. Each chunk is recursively divided across 3 depth levels ($4 \times 4 \times 4$), producing **64 distinct binary blocks**[cite: 1, 2].
3. Each block is individually encrypted sequentially with **Password 2** and **Password 3**[cite: 1, 2].

### 3. Permutation & Container Packaging
1. The 64 encrypted blocks undergo a randomized **Fisher-Yates shuffle**[cite: 1, 2].
2. A `manifest_tree.dat` file is created containing original metadata and the permutation mapping array[cite: 1, 2].
3. The shuffled blocks (`/blocks/block_N.bin`) and manifest are packaged into a `.m22c` archive via JSZip[cite: 1, 2].

---

## Container Specification (`.m22c`)

An `.m22c` container is a ZIP-structured archive containing the following layout:

```
file_name.m22c/
├── manifest_tree.dat
└── blocks/
    ├── block_0.bin
    ├── block_1.bin
    ├── ...
    └── block_63.bin
```

---

## Tech Stack & Requirements

* **Frontend**: HTML5, CSS3, ES6+ JavaScript[cite: 2]
* **Cryptography**: Native Web Crypto API (`crypto.subtle`)[cite: 1, 2]
* **Archiving Library**: [JSZip](https://stuk.github.io/jszip/) (v3.10.1)[cite: 1]
* **Browser Requirements**: Modern Chromium-based browser (optimized for Google Chrome)[cite: 2]

---

## Usage Instructions

1. Open `index.html` in your web browser.
2. Drag and drop a file into the drop zone (or click to select)[cite: 2].
3. Enter valid passwords for Level 1, Level 2, and Level 3 encryption.
4. Click **Avvia Pipeline di Cifratura M22C** to process the file.
5. Download the resulting `.m22c` file to your local system.

---

## Future Development Roadmap

* [ ] Implement complete Decryption Engine (Manifest parsing + reverse Fisher-Yates permutation + 3-pass decryption).
* [ ] Integrate Web Workers to offload heavy cryptographic operations for large files (> 50 MB).
* [ ] Implement optional image camouflage / steganography container packaging.

---

## License & Credits

* **Author**: Powered by [Benevolo B.](https://cruelben.github.io/)
* **License**: Released under the **Open Surge License**.