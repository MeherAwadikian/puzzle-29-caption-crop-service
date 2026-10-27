# Puzzle #29 — image captioning/cropping FastAPI service, callable from n8n

Brief (Task Forge puzzle tracker): image captioning/cropping FastAPI service, callable from n8n — Python (FastAPI) + n8n + Google Cloud Run.

## What's real here

- **`app.py`** — FastAPI service, `POST /process` (multipart image upload):
  1. **Crop** — `energy_crop()` crops to a requested aspect ratio by sliding a window
     across a Sobel-edge-detail map and keeping the highest-energy position, instead
     of a naive dead-center crop. Verified live against a deliberately off-center
     synthetic test image (red subject at the right edge of a dark frame) — the
     returned crop centers on the subject (`cropped_out.jpg` in this repo is the
     real output from that run).
  2. **Caption** — shells out to the already-authenticated `claude` CLI
     (`claude -p "Read the image file at <path>..." --allowedTools Read`), a real
     multimodal model call, not a canned string. Confirmed OpenRouter's current free
     vision models (`google/gemini-2.0-flash-exp:free`,
     `meta-llama/llama-3.2-11b-vision-instruct:free`) are both dead/404 before
     switching to this approach.
  3. Returns `{caption, caption_available, cropped_width, cropped_height,
     cropped_image_base64, cropped_image_mime}` as one JSON response.
- **`n8n-workflow.json`** — Webhook → HTTP Request (multipart POST to the service) →
  Code node (unpacks the JSON + base64 image into n8n binary data) → Respond to
  Webhook. Written by hand against the documented n8n node schemas (HTTP Request
  v4.2, Code v2, Respond to Webhook v1.2) — **not yet validated against n8n-mcp or
  imported into the live container**, see Blockers below.

## Verified live (this session)

```
POST /process?target_width=400&target_height=400  (test_offcenter.jpg)
→ caption: "A red circle sitting atop a red rectangle, resembling a simple
            robot or icon shape, on a dark navy background."
→ cropped_width/height: 400/400
→ cropped_image_base64: real JPEG, decodes to cropped_out.jpg (subject intact,
   not clipped — proves the energy crop isn't a dumb center crop)
```

## Blockers (documented, not worked around)

1. **Local Docker n8n container unreachable this session.** Docker Desktop's WSL
   disk is a symlink to `D:\Docker\wsl\disk`, and the D: drive isn't currently
   attached to this machine (only C:/E: present) — a disk-relocation issue from
   an earlier session's cleanup, unrelated to this puzzle. Repointing the symlink
   or reattaching D: is a system-level change outside this task's scope to do
   blind; `n8n-workflow.json` is ready to `n8n import:workflow` the moment the
   container is reachable again.
2. **No Google Cloud account/`gcloud` CLI on this machine** (confirmed, not
   assumed) — same "named tool not actually provisioned" pattern as most of this
   puzzle series (Bolt.new, Lovable, VAPI, Voiceflow credits). Substituted
   Cloud Run with **Fly.io** (flyctl is installed) for the real deploy target, but
   `fly auth login` needs an interactive browser step this session can't do
   headlessly — same class of blocker as puzzle #10's Vercel/GitHub connection.
3. **Captioning is intentionally local-only in the Docker image** — baking this
   machine's authenticated Claude CLI session into a publicly deployed container
   would leak live credentials, so the public image ships crop-only; `caption`
   comes back `null`/`caption_available: false` when the CLI isn't present. This
   is a deliberate security choice, not an oversight.

## To finish once unblocked
- Reattach D: (or repoint the Docker Desktop WSL disk symlink) → start Docker →
  `docker cp n8n-workflow.json n8n:/tmp/` → `docker exec n8n n8n import:workflow
  --input=/tmp/n8n-workflow.json` → open in browser, trigger the webhook with a
  real multipart image POST, confirm the response end-to-end.
- `fly auth login` (interactive) → `fly launch --no-deploy` → `fly deploy` from
  this directory → public crop-only endpoint live.
