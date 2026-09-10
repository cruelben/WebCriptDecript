# M22C Secure Suite v2

M22C Secure Suite is a client-side web application for local file
protection using AES-256-GCM encryption, PBKDF2-SHA-256 key derivation,
and recursive fragmentation into 64 logical fragments.

The project is designed to operate without an M22C application server:
the selected file is processed in the browser and the M22C containers
are generated locally.

> **Important:** M22C is an experimental/local protection project.
> Encryption should not be interpreted as an absolute guarantee against
> every form of metadata analysis, device compromise, password loss, or
> forensic analysis.

## Main features

- AES-256-GCM multi-level encryption.
- PBKDF2-SHA-256 key derivation.
- 600,000 PBKDF2 iterations for format v2.
- Legacy v1 read compatibility with 210,000 iterations.
- Recursive **4 × 4 × 4 = 64 logical fragments**.
- Random distribution of fragments across physical containers.
- Dynamic number of containers based on the configured maximum size.
- Minimum of 3 physical containers.
- ZIP-based `.m22c` containers.
- Encrypted `manifest_tree.dat`.
- SHA-256 integrity verification.
- Random, ordered, or custom container naming.
- Optional JPG camouflage.
- Optional best-effort metadata removal before encryption.
- File System Access API support when available.
- Browser-download fallback.
- Optional deletion of original containers after a verified recovery.

## Encryption pipeline

``` text
ORIGINAL FILE
      │
      ├── optional: try to remove metadata
      │
      ▼
M22X + data
      │
      ▼
AES-256-GCM / Password 1
      │
      ▼
4 sections
      │
      ├── AES-256-GCM / Password 2
      │
      ▼
16 sections
      │
      ├── AES-256-GCM / Password 3
      │
      ▼
64 logical fragments
      │
      ▼
cryptographic shuffle
      │
      ▼
physical distribution
      │
      ▼
3 or more .m22c containers
```

## Three passwords

M22C uses three independent password fields:

``` text
Password 1
Password 2
Password 3
```

The manifest password is derived from:

``` text
Password1_Password2_Password3
```

The interface currently contains test defaults:

``` text
123
456
789
```

These values are intended only for testing and must not be used for real
protected data.

## Cryptography

Format v2 uses:

``` text
AES-256-GCM
PBKDF2-SHA-256
600,000 iterations
16-byte salt
12-byte AES-GCM IV
256-bit AES keys
```

Salt and IV values are generated with the browser’s cryptographically
secure random generator.

Application-level shuffling uses Fisher-Yates with rejection sampling to
avoid modulo bias.

## Encrypted manifest

Each container set contains:

``` text
manifest_tree.dat
```

The manifest is encrypted and contains the information required to
reconstruct the original encrypted payload, including:

- format version;
- algorithm and KDF parameters;
- original filename;
- original payload size;
- SHA-256;
- ordered list of the 64 logical fragments;
- metadata-sanitization information when requested.

The manifest is essential because it defines the exact logical
reconstruction order.

## Dynamic containers

The user can configure:

``` text
Maximum container size (MB)
```

The default is:

``` text
10 MB
```

Rules:

- at least 3 containers are generated;
- larger files may require more containers;
- decryption accepts a variable number of containers;
- physical parts are validated against the manifest;
- duplicate, missing, and undeclared fragments are rejected.

The hard maximum configurable container limit is 4096 MB.

## Physical fragment parts

When required by the configured physical container size, a logical
fragment may be split into physical parts such as:

``` text
sec_ab12_01__part_001.bin
sec_ab12_01__part_002.bin
sec_ab12_01__part_003.bin
```

The parts are reconstructed during decryption before the logical
64-fragment tree is rebuilt.

## SHA-256 integrity

For format v2, SHA-256 is calculated over the payload that is actually
encrypted.

With metadata removal disabled:

``` text
original file
     ↓
SHA-256
     ↓
encryption
```

When metadata removal is requested and succeeds:

``` text
original file
     ↓
best-effort metadata removal
     ↓
sanitized in-memory payload
     ↓
SHA-256
     ↓
encryption
```

After reconstruction, the hash and payload size are checked before the
recovered file is delivered.

## Try to remove metadata before encryption

The encryption interface provides:

``` text
☐ Try to remove metadata before encryption
```

The option is **disabled by default**.

The wording is deliberately conservative: M22C does not promise complete
metadata removal.

The operation is best-effort and currently targets supported image
formats.

### JPEG

The sanitizer can attempt to remove common metadata segments such as:

- EXIF;
- XMP;
- IPTC / Photoshop IRB;
- JPEG comments.

The compressed image data is not re-encoded solely for this operation.

### PNG

The sanitizer can remove selected metadata-related chunks, including:

``` text
eXIf
tEXt
zTXt
iTXt
tIME
pHYs
cHRM
gAMA
sRGB
iCCP
```

Other chunks are preserved.

### Unsupported formats

For unsupported formats, M22C does not perform generic destructive
rewriting.

The original bytes are encrypted unchanged and the log reports that
automatic metadata sanitization is not supported.

### Original file is never modified

Metadata processing operates on data held in memory.

The original file selected by the user is not overwritten or modified on
disk.

If metadata removal succeeds, the recovered file can therefore differ
byte-for-byte from the original because supported metadata may have been
removed.

## Metadata privacy limitations

Removing metadata from the original payload does **not** make the entire
operation anonymous.

The M22C containers are ZIP archives and may expose structural
information such as:

- the fact that the file is a ZIP archive;
- internal filenames;
- number of entries;
- entry sizes;
- ZIP timestamps or other structural fields.

Filesystem metadata outside the container may also exist, such as the
container’s filesystem creation/modification times and the chosen
container filename.

Therefore:

``` text
content encryption
≠
complete metadata removal
```

The optional metadata feature is a privacy-enhancement attempt, not a
guarantee of anonymization.

## JPG camouflage

M22C can optionally append an M22C payload to a JPEG image.

The JPEG remains normally viewable while M22C can detect the appended
payload using a dedicated marker and length information.

Available source modes include:

- automatically generated images;
- one image reused for all containers;
- one image per container.

The final JPEG is larger than the underlying `.m22c` payload because it
also contains the image itself.

Camouflage is disabled by default.

## Container naming

Three naming styles are available.

### Random

``` text
a8k3m1xz.m22c
q9f2p7ab.m22c
```

### Ordered

``` text
M22C-01.m22c
M22C-02.m22c
M22C-03.m22c
```

### Custom prefix

``` text
PROJECT-01.m22c
PROJECT-02.m22c
PROJECT-03.m22c
```

Random names can reduce obvious identification of the container set,
although filenames themselves are not a cryptographic security boundary.

## Decryption

The recovery pipeline is the reverse operation:

``` text
M22C / camouflage JPEG
          ↓
container extraction
          ↓
manifest recovery
          ↓
manifest validation
          ↓
physical fragment reconstruction
          ↓
L3 reconstruction + Password 3
          ↓
L2 reconstruction + Password 2
          ↓
L1 reconstruction + Password 1
          ↓
M22X verification
          ↓
size verification
          ↓
SHA-256 verification
          ↓
RIP_original-name
```

The application rejects invalid or incomplete container sets.

## Browser support

M22C uses:

- HTML5;
- modern JavaScript;
- Web Crypto API;
- File API;
- File System Access API when supported;
- JSZip 3.10.1;
- Tailwind CSS via CDN.

When File System Access API is unavailable, the application falls back
to standard browser downloads.

Automatic deletion of original containers is only available when the
required filesystem permissions are supported.

## Client-side model

The application is designed to process selected data locally in the
browser without an M22C backend.

Actual security also depends on:

- browser security;
- operating-system security;
- installed extensions;
- malware on the device;
- password quality and storage;
- the way the HTML file and its dependencies are obtained.

For high-security deployments, consider locally hosting and verifying
dependencies instead of relying on third-party CDNs.

## Project structure

The project is intentionally distributed as a single HTML file:

``` text
index.html
```

Logical components include:

``` text
UI
├── Encryption tab
├── Decryption tab
├── Password controls
├── Container-size controls
├── Naming controls
├── Metadata-sanitization option
├── JPG camouflage controls
├── Output-folder controls
├── Progress bar
└── Status log

Crypto
├── PBKDF2
├── AES-256-GCM
├── SHA-256
├── Secure randomness
└── Fisher-Yates shuffle

Fragmentation
├── L1
├── L2
├── L3
└── 64 logical fragments

Packaging
├── JSZip
├── Dynamic containers
├── Physical parts
└── JPG camouflage

Decryption
├── Extraction
├── Manifest validation
├── Fragment validation
├── Reconstruction
├── Integrity verification
└── Recovery
```

## Validation and error handling

The decryption process validates, among other things:

- minimum container count;
- manifest presence;
- manifest version;
- algorithm;
- KDF parameters;
- exactly 64 logical fragments;
- duplicate fragments;
- missing fragments;
- undeclared fragments;
- corrupted physical parts;
- incorrect passwords;
- `M22X` header;
- reconstructed payload size;
- SHA-256 integrity.

## Testing checklist

Recommended tests include:

1.  Encrypt and recover a small text file.
2.  Encrypt a JPEG with metadata removal disabled.
3.  Encrypt the same JPEG with metadata removal enabled.
4.  Test a PNG containing metadata.
5.  Test an unsupported file format.
6.  Test multiple maximum-container-size values.
7.  Test dynamic sets containing more than 3 containers.
8.  Test JPG camouflage and recovery.
9.  Test incorrect passwords.
10. Test a missing or corrupted container.

## Security notes

M22C should be treated as an experimental protection tool rather than a
replacement for a mature, independently audited encryption product.

In particular:

- use strong, unique passwords;
- never rely on the default test passwords for real data;
- keep backup copies of important data;
- do not assume that encrypted container metadata is invisible;
- remember that losing the required passwords makes recovery impossible;
- test the exact browser and environment used for deployment.

## Attribution

**Powered by Benevolo B.**

Project website:

https://cruelben.github.io/

## License

No license is implied by this README.

Add an explicit license file to the repository if the project is
intended to be redistributed, modified, or reused by third parties.

## Project status

**M22C Secure Suite v2**

Current feature set:

- AES-256-GCM
- PBKDF2-SHA-256 — 600,000 iterations
- 64 logical fragments
- dynamic containers
- minimum 3 containers
- encrypted manifest
- SHA-256 end-to-end integrity verification
- optional JPG camouflage
- legacy v1 read compatibility
- optional best-effort metadata removal
- client-side processing
- File System Access API + browser-download fallback
