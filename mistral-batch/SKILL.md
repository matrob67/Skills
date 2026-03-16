---
name: mistral-batch
description: How to embed large datasets (100k+ chunks) efficiently using the Mistral Batch Embeddings API. Use this skill when you need to embed more than a few thousand texts with mistral-embed, want to avoid rate limits and per-call overhead, or need to process a large corpus overnight. Trigger when the user mentions batch embedding, JSONL upload to Mistral, submitting batch jobs, polling job status, or collecting embeddings results into a FAISS index.
---

# Mistral Batch Embeddings API

## Concept
Instead of calling `client.embeddings.create()` one batch at a time (slow, rate-limited), you:
1. Prepare a JSONL file with all requests
2. Upload it and submit a batch job
3. Poll until done (minutes to hours)
4. Download the output JSONL and parse embeddings

**Best for:** 50k+ chunks. Below that, the real-time API is faster.

## JSONL Format

Each line must be a valid JSON object:
```json
{"custom_id": "doc_001__chunk_0", "body": {"model": "mistral-embed", "input": "text to embed..."}, "_meta": {"decision_key": "doc_001", "chunk_idx": 0}}
```

**Important:** `_meta` is a custom field for your own tracking — strip it before sending to Mistral (include it in the JSONL for your own use but don't put it inside `body`).

### Constraints
- Max **50,000 requests per JSONL file** → split into multiple files if needed
- Each input max **8,192 tokens**
- Use `cl100k_base` tiktoken encoding to estimate tokens

## Step 1 — Prepare JSONL batches

```python
import json, tiktoken
from pathlib import Path

enc = tiktoken.get_encoding("cl100k_base")
MAX_TOKENS = 8000
BATCH_SIZE = 50_000

def count_tokens(text):
    if len(text) > 100_000:
        return len(text) // 4  # fast path
    return len(enc.encode(text))

def truncate(text):
    if count_tokens(text) <= MAX_TOKENS:
        return text
    text = text[:MAX_TOKENS * 2]
    if count_tokens(text) > MAX_TOKENS:
        text = text[:MAX_TOKENS]
    return text

chunks = [...]  # list of {"custom_id": "...", "text": "...", "meta": {...}}
batch_files = []
batch_idx = 0
lines = []

for chunk in chunks:
    text = truncate(chunk["text"])
    line = {
        "custom_id": chunk["custom_id"],
        "body": {"model": "mistral-embed", "input": text},
        "_meta": chunk["meta"]
    }
    lines.append(json.dumps(line, ensure_ascii=False))

    if len(lines) >= BATCH_SIZE:
        path = Path(f"data/batch_{batch_idx:04d}.jsonl")
        path.write_text("\n".join(lines), encoding="utf-8")
        batch_files.append(path)
        lines = []
        batch_idx += 1

if lines:
    path = Path(f"data/batch_{batch_idx:04d}.jsonl")
    path.write_text("\n".join(lines), encoding="utf-8")
    batch_files.append(path)
```

## Step 2 — Upload and submit jobs

```python
from mistralai import Mistral
import os

client = Mistral(api_key=os.getenv("MISTRAL_API_KEY"))

def submit_jobs(batch_files):
    jobs = []
    for batch_file in batch_files:
        print(f"Uploading {batch_file.name} ({batch_file.stat().st_size/1024/1024:.1f} MB)...")

        # Correct upload format for mistralai SDK
        with open(batch_file, "rb") as f:
            uploaded = client.files.upload(
                file={"file_name": batch_file.name, "content": f},
                purpose="batch"
            )

        job = client.batch.jobs.create(
            input_files=[uploaded.id],
            model="mistral-embed",
            endpoint="/v1/embeddings",
            metadata={"batch_file": str(batch_file)}
        )
        jobs.append({
            "job_id": job.id,
            "file_id": uploaded.id,
            "batch_file": str(batch_file),
            "status": job.status
        })
        print(f"  Submitted: {job.id} ({job.status})")
    return jobs
```

## Step 3 — Poll status

```python
import time

def poll_jobs(jobs, interval=60):
    while True:
        pending = [j for j in jobs if j["status"] not in ("SUCCESS", "FAILED")]
        if not pending:
            break

        print(f"[{time.strftime('%H:%M:%S')}] Checking {len(pending)} jobs...")
        for j in pending:
            job = client.batch.jobs.get(job_id=j["job_id"])
            j["status"] = job.status
            if job.status == "SUCCESS":
                j["output_file_id"] = job.output_file
            n = job.total_requests or 1
            done = job.succeeded_requests or 0
            print(f"  {job.id[:8]}: {job.status} {done:,}/{n:,} ({done/n*100:.0f}%)")

        still_pending = [j for j in jobs if j["status"] not in ("SUCCESS", "FAILED")]
        if not still_pending:
            break
        time.sleep(interval)

    print("All jobs done.")
    return jobs
```

## Step 4 — Collect results into FAISS

```python
import numpy as np
import faiss

def collect_results(jobs, meta_lookup):
    """
    meta_lookup: dict {custom_id -> metadata dict}
    """
    index = faiss.IndexFlatIP(1024)  # cosine similarity with normalized vectors
    all_meta = []

    for job_info in jobs:
        if job_info["status"] != "SUCCESS":
            continue

        out_file_id = job_info["output_file_id"]
        result_bytes = client.files.download(file_id=out_file_id).read()
        lines = result_bytes.decode("utf-8").strip().split("\n")

        vecs = []
        metas = []
        for line in lines:
            if not line.strip():
                continue
            r = json.loads(line)
            emb = r.get("response", {}).get("body", {}).get("data", [{}])[0].get("embedding")
            if not emb:
                continue
            cid = r["custom_id"]
            vecs.append(emb)
            metas.append(meta_lookup.get(cid, {"custom_id": cid}))

        if vecs:
            vecs_np = np.array(vecs, dtype=np.float32)
            faiss.normalize_L2(vecs_np)  # normalize for cosine similarity
            index.add(vecs_np)
            all_meta.extend(metas)
            print(f"  +{len(vecs):,} vectors | Total: {index.ntotal:,}")

    return index, all_meta
```

## Pricing
- Batch API is typically **50% cheaper** than real-time API
- Jobs run asynchronously, results available within hours
- No rate limit issues since it's async

## State management (resumable)
Save job state to JSON so you can resume if the script is interrupted:

```python
state_file = Path("data/batch_state.json")

def save_state(jobs):
    state_file.write_text(json.dumps({"jobs": jobs}, indent=2))

def load_state():
    if state_file.exists():
        return json.loads(state_file.read_text())["jobs"]
    return []
```

## Atomic file saves
Always save FAISS index and metadata atomically to prevent corruption:

```python
def atomic_save(index, all_meta, index_path, meta_path):
    # Save index
    tmp = index_path.with_suffix(".bin.tmp")
    faiss.write_index(index, str(tmp))
    tmp.replace(index_path)

    # Save metadata
    tmp2 = meta_path.with_suffix(".json.tmp")
    tmp2.write_text(json.dumps(all_meta), encoding="utf-8")
    tmp2.replace(meta_path)
```

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| `ValidationError: Input should be valid dict or File` | Wrong upload format | Use `{"file_name": ..., "content": file_obj}` dict |
| `Input has X tokens, exceeding max 8192` | Chunk too long | Truncate before building JSONL |
| `UnicodeEncodeError cp1252` | Emoji/special chars in print() on Windows | Replace `✅`, `✓` etc. with ASCII in print statements |
| Job stuck in QUEUED | Mistral queue busy | Normal — just wait, poll every 60s |
