# GSE — Genome Storage Engine

One storage engine for every service you ship.

GSE is Quantumhash's internal object store: every file is content-addressed,
encrypted before it touches disk, and automatically deduplicated. Upload
once, get a permanent URL back, and never think about buckets, ACLs, or
which region a file lives in again.

**Base URL:** `https://gse.quantumhash.me`

```bash
# one request, one URL back
curl -H "X-API-Key: $GSE_API_KEY" \
  -F "file=@invoice.pdf" \
  -F "service_name=billing-service" \
  https://gse.quantumhash.me/v1/uploads
```

```json
{
  "content_id": "a1c9...f0e2",
  "deduplicated": false,
  "stored_size": 18320,
  "service_name": "billing-service"
}
```

The file is now live at:

```
https://gse.quantumhash.me/v1/objects/a1c9...f0e2/content?service_name=billing-service
```

---

## Table of contents

- [Why teams choose GSE](#why-teams-choose-gse)
- [How a file is stored](#how-a-file-is-stored)
- [Content IDs, explained](#content-ids-explained)
- [Quickstart](#quickstart)
- [API reference](#api-reference)
- [Upload a file — by language](#upload-a-file--by-language)
  - [cURL](#curl)
  - [Node.js](#nodejs)
  - [Python](#python)
  - [PHP](#php)
  - [Go](#go)
- [Serving files back](#serving-files-back)
- [Security model](#security-model)
- [Limits & behaviour](#limits--behaviour)
- [FAQ](#faq)

---

## Why teams choose GSE

Six things it does that a raw S3 bucket doesn't.

**Storage that doesn't repeat itself**
Every upload is fingerprinted with BLAKE3. Upload the same file from three
different services and it's stored once — GSE just adds a reference.
Re-uploading a 500 MB recording five times costs the storage of one.

**Encrypted before it's written**
Every object is encrypted with XChaCha20-Poly1305, keyed through HashiCorp
Vault. Your application never handles a key — GSE asks Vault to wrap and
unwrap them on every request.

**Large files, handled quietly**
Anything over 1 MB is split into content-defined chunks and uploaded in
parallel. A 2 GB file that took three minutes on the first pass now takes
about thirty seconds — you don't change anything to get this.

**Your files, your bucket**
Each service gets its own storage bucket, named after it. Nothing from
`billing-service` is reachable through a `reports-service` URL, even by
accident.

**Play it, don't just download it**
Audio and video are served with correct content types and HTTP range
support, so a browser's built-in player can seek to any point instead of
forcing a download.

**One URL, no expiry**
The link GSE gives you back is permanent — no signing, no fifteen-minute
timer to work around. Store it in your database once and use it for the
life of the record.

---

## How a file is stored

What happens between your upload request and the response:

1. **Fingerprint** — BLAKE3 hashes the file. This hash becomes the
   `content_id`, the file's permanent identity.
2. **Check for a match** — if that exact content already exists for your
   service, GSE stops here and hands back the existing ID.
3. **Chunk & compress** — large files are split into pieces; compressible
   content is shrunk before it's encrypted.
4. **Encrypt** — each piece is sealed with a key wrapped by Vault. Nothing
   is written to disk unencrypted.
5. **Store & confirm** — bytes land in your service's bucket; metadata is
   committed to the database, and the response returns.

Reading a file runs the same sequence in reverse: locate the metadata,
fetch the encrypted pieces, decrypt, decompress, reassemble.

---

## Content IDs, explained

The one thing every integration gets wrong on the first try.

Every upload response includes three identifiers. Only one of them belongs
in a URL.

| Field | Looks like | What it's for |
|---|---|---|
| `content_id` | `a1c9b7...f0e2` (64 chars) | The file's identity. Use this in every download or metadata URL. |
| `object_id` | `e0be5428-078b-...` | Internal record ID. Not resolvable through the API — don't put it in a URL. |
| `file_id` | `daf4ca9e-90de-...` | Identifies this specific upload event, for your own audit trail. |

> **The most common integration bug**
> Building a download URL from `object_id` instead of `content_id` returns
> `404 Object not found` — it looks like a missing file, but it's the wrong
> identifier. If a URL that was generated straight from an upload response
> 404s, check which ID went into it first.

---

## Quickstart

Three requests: upload, confirm, retrieve.

### 1. Get a service name and API key

Ask your platform team for an API key and agree on a `service_name` — a
short, stable slug for your service (e.g. `billing-service`,
`meeting-bot-storage`). The first upload with a new name creates its
bucket automatically; nothing needs provisioning up front.

### 2. Upload

```bash
curl -H "X-API-Key: $GSE_API_KEY" \
  -F "file=@report.pdf" \
  -F "service_name=billing-service" \
  https://gse.quantumhash.me/v1/uploads
```

### 3. Build the file's URL

Take `content_id` from the response and combine it with your
`service_name`. This URL is permanent — save it in your database.

```
https://gse.quantumhash.me/v1/objects/{content_id}/content?service_name={service_name}
```

### 4. Retrieve it — from anywhere, no key required

```bash
curl -o report.pdf \
  "https://gse.quantumhash.me/v1/objects/a1c9...f0e2/content?service_name=billing-service"
```

---

## API reference

Base URL: `https://gse.quantumhash.me`

| Endpoint | Auth | Purpose |
|---|---|---|
| `GET /v1/health` | none | Liveness check. |
| `POST /v1/uploads` | API key | Upload a file. Multipart form: `file`, `service_name`. |
| `GET /v1/objects/{content_id}` | API key | Metadata for a file: size, compression, chunked status, state. |
| `GET /v1/objects/{content_id}/content` | none | Download or stream the file itself. Supports HTTP range requests. |

### `POST /v1/uploads`

Send as `multipart/form-data`.

| Field | Required | Description |
|---|---|---|
| `file` | yes | The file being uploaded, any type or size. |
| `service_name` | yes | Your service's slug. Determines which bucket the file lands in. |

**201 response**

```json
{
  "content_id": "a1c9b7...f0e2",
  "object_id": "e0be5428-078b-4791-8...",
  "file_id": "daf4ca9e-90de-4e8d-8...",
  "chunked": false,
  "deduplicated": false,
  "plaintext_size": 84861,
  "stored_size": 18320,
  "service_name": "billing-service"
}
```

### `GET /v1/objects/{content_id}/content`

Returns the decrypted, decompressed file with a correct `Content-Type` and
`Content-Disposition: inline`, so browsers render PDFs, images, and audio
in place instead of downloading them. Requires `?service_name=` as a query
parameter — it must match the service the file was uploaded under, or the
request 404s.

---

## Upload a file — by language

The same request, in five languages. Replace the placeholder key and
service name with your own.

### cURL

```bash
curl -H "X-API-Key: $GSE_API_KEY" \
  -F "file=@/path/to/file.pdf" \
  -F "service_name=your-service-name" \
  https://gse.quantumhash.me/v1/uploads
```

### Node.js

```js
// npm install form-data node-fetch
const FormData = require('form-data');
const fetch = require('node-fetch');
const fs = require('fs');

const GSE_API_URL = process.env.GSE_API_URL;   // https://gse.quantumhash.me
const GSE_API_KEY = process.env.GSE_API_KEY;
const GSE_SERVICE_NAME = process.env.GSE_SERVICE_NAME;

async function uploadToGSE(filePath, originalName) {
  const form = new FormData();
  form.append('file', fs.createReadStream(filePath), originalName);
  form.append('service_name', GSE_SERVICE_NAME);

  const res = await fetch(`${GSE_API_URL}/v1/uploads`, {
    method: 'POST',
    headers: { 'X-API-Key': GSE_API_KEY },
    body: form,
  });
  if (!res.ok) throw new Error(`GSE upload failed: ${res.status}`);

  const result = await res.json();
  result.url = `${GSE_API_URL}/v1/objects/${result.content_id}/content?service_name=${GSE_SERVICE_NAME}`;
  return result; // { content_id, url, deduplicated, ... }
}

module.exports = { uploadToGSE };
```

### Python

```python
# pip install requests
import os
import requests

GSE_API_URL = os.environ["GSE_API_URL"]
GSE_API_KEY = os.environ["GSE_API_KEY"]
GSE_SERVICE_NAME = os.environ["GSE_SERVICE_NAME"]


def upload_to_gse(file_path: str) -> dict:
    with open(file_path, "rb") as f:
        resp = requests.post(
            f"{GSE_API_URL}/v1/uploads",
            headers={"X-API-Key": GSE_API_KEY},
            files={"file": f},
            data={"service_name": GSE_SERVICE_NAME},
        )
    resp.raise_for_status()
    result = resp.json()
    result["url"] = (
        f"{GSE_API_URL}/v1/objects/{result['content_id']}"
        f"/content?service_name={GSE_SERVICE_NAME}"
    )
    return result
```

### PHP

```php
<?php

function upload_to_gse(string $filePath, string $originalName): array {
    $apiUrl = getenv('GSE_API_URL');
    $apiKey = getenv('GSE_API_KEY');
    $serviceName = getenv('GSE_SERVICE_NAME');

    $ch = curl_init("$apiUrl/v1/uploads");
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ["X-API-Key: $apiKey"],
        CURLOPT_POSTFIELDS => [
            'file' => new CURLFile($filePath, mime_content_type($filePath), $originalName),
            'service_name' => $serviceName,
        ],
    ]);

    $response = curl_exec($ch);
    $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($status >= 300) {
        throw new Exception("GSE upload failed: $status");
    }

    $result = json_decode($response, true);
    $result['url'] = "$apiUrl/v1/objects/{$result['content_id']}/content?service_name=$serviceName";
    return $result;
}
```

### Go

```go
package gse

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"mime/multipart"
	"net/http"
	"os"
)

type UploadResult struct {
	ContentID    string `json:"content_id"`
	Deduplicated bool   `json:"deduplicated"`
	URL          string `json:"-"`
}

func UploadToGSE(filePath string) (*UploadResult, error) {
	apiURL := os.Getenv("GSE_API_URL")
	apiKey := os.Getenv("GSE_API_KEY")
	serviceName := os.Getenv("GSE_SERVICE_NAME")

	file, err := os.Open(filePath)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	var buf bytes.Buffer
	w := multipart.NewWriter(&buf)
	part, _ := w.CreateFormFile("file", filePath)
	io.Copy(part, file)
	w.WriteField("service_name", serviceName)
	w.Close()

	req, _ := http.NewRequest("POST", apiURL+"/v1/uploads", &buf)
	req.Header.Set("X-API-Key", apiKey)
	req.Header.Set("Content-Type", w.FormDataContentType())

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 300 {
		return nil, fmt.Errorf("gse upload failed: %d", resp.StatusCode)
	}

	var result UploadResult
	json.NewDecoder(resp.Body).Decode(&result)
	result.URL = fmt.Sprintf("%s/v1/objects/%s/content?service_name=%s", apiURL, result.ContentID, serviceName)
	return &result, nil
}
```

---

## Serving files back

Point at it directly — GSE handles the rest.

Once you have a URL, use it exactly like any static asset URL: in an
`<img>`, an `<a>`, an audio player, or handed to a third-party API that
needs to fetch the file itself (an OCR service, for instance). No signing
step, no extra request to "unlock" it first.

**Images & PDFs** — served with the correct MIME type and `inline`
disposition, so they render in the browser rather than downloading.

**Audio & video** — range requests are supported, so native players can
seek anywhere in the file instead of only playing from the start.

> **Handing a URL to a third party**
> Because download URLs need no API key, they're safe to pass directly to
> external services that fetch content on your behalf — an OCR pipeline, a
> transcription job, a preview generator. There's nothing for them to
> authenticate.

---

## Security model

What's protected, and why downloads don't need a key.

**Uploads are locked down.** Every write goes through `X-API-Key`. Without
a valid key, nothing can be written to any bucket.

**Downloads are open by design — with three guards.** Making reads public
removes a whole class of integration friction (expiring links, proxy
servers, signature mismatches). It's safe because of three separate
properties, not one:

- **Bucket isolation** — a file lives only inside its own service's
  bucket. Requesting it with any other `service_name` resolves to
  nothing — a clean 404, not a permission error that reveals the file
  exists.
- **Unguessable IDs** — `content_id` is a 64-character cryptographic
  hash. There's no sequential ID, no predictable pattern to walk through.
- **No listing** — there is no endpoint that lists what's in a bucket.
  Knowing a `content_id` is the only way in — you can't browse to find
  one.
- **Encrypted at rest either way** — every object is still encrypted on
  disk. A public download endpoint decrypts on the way out; the
  underlying storage is never plaintext.

> **If a file genuinely needs access control**
> Put GSE's URL behind your own authenticated endpoint and fetch it
> server-side — your app checks the user, then streams the bytes through.
> GSE's public-read model assumes "has the link" is an acceptable bar; if
> it isn't for a given file, add that check in your own service rather
> than in GSE.

---

## Limits & behaviour

| Behaviour | Detail |
|---|---|
| Chunking threshold | Files ≥ 1 MB are automatically split and uploaded in parallel; smaller files are stored whole. |
| Deduplication scope | Per service. Identical content uploaded twice under the same `service_name` is stored once; different services each keep their own copy. |
| Compression | Applied automatically based on file type — text and documents are compressed; already-compressed media (audio, video, images) is stored as-is. |
| Re-uploading the same file | Returns the existing `content_id` with `"deduplicated": true` — no new bytes are written, but a new upload record is still logged. |
| Deleting a file | Not exposed to clients directly. Files are cleaned up automatically once nothing references them — talk to the platform team if you need something removed sooner. |

---

## FAQ

**My download returns "Object not found" right after uploading.**
Almost always an ID mix-up — check that the URL uses `content_id`, not
`object_id` or `file_id`. See [Content IDs, explained](#content-ids-explained).

**Do I need to provision a bucket before my first upload?**
No. The first upload with a new `service_name` creates its bucket
automatically.

**Can I use the same `service_name` across environments (staging, prod)?**
Don't — use a distinct name per environment (e.g.
`billing-service-staging`) so files, and their dedup savings, stay
separate.

**Is there a file size limit?**
Large files are supported and chunked automatically. If you're regularly
moving multi-gigabyte files, check with the platform team so upstream
proxy limits are sized to match.

**What happens if I upload the exact same file twice?**
You get the same `content_id` back both times, with
`"deduplicated": true` on the second response. No extra storage is used.

---

GSE — Genome Storage Engine · Internal service, Quantumhash Corporation
