## Mobile ↔ MagicQuill Backend Guide

This document explains how a Kotlin or iOS client can talk to the existing FastAPI backend (currently bundled with the Gradio UI) without modifying the Python code. It focuses on HTTP endpoints, expected payloads, and recommended client flows.

---

### 1. High-Level Architecture

- **Mobile client** captures images, masks, and textual prompts locally.
- **HTTP API** (FastAPI in `gradio_run.py`) exposes endpoints under `http://<server>:7860`.
- **Model worker** runs on the server and performs prompt guessing, preprocessing, and image generation (via `LLaVAModel` and `ScribbleColorEditModel`).

For production use, host the FastAPI app separately from Gradio by mounting the existing app (as already done with `gr.mount_gradio_app`). Mobile apps communicate directly with the FastAPI routes.

---

### 2. Existing Endpoints

#### 2.1 POST `/magic_quill/guess_prompt`

Purpose: Ask the LLaVA model to infer prompt suggestions based on user-provided imagery.

**Request JSON**
```json
{
  "original_image": "data:image/png;base64,....",
  "add_color_image": "data:image/png;base64,....",
  "add_edge_image": "data:image/png;base64,...."
}
```
- Images must be base64 strings (PNG/JPEG/WebP allowed) with data URI prefixes.
- `add_color_image` can be omitted (set equal to `original_image`).
- `add_edge_image` can be omitted; the backend builds an empty mask if missing.

**Response**
```json
"coat of arms, fantasy emblem"
```
Plain-text response listing comma-separated prompt ideas. The client should treat it as an opaque string.

#### 2.2 POST `/magic_quill/process_background_img`

Purpose: Resize or normalize a background image before further processing.

**Request JSON**
```json
"data:image/png;base64,...."
```
Single string: the base64-encoded image. The backend uses the globally configured resolution (`RES`) to resize the smaller dimension.

**Response JSON**
```json
"data:image/png;base64,...."
```
Resized image returned as a PNG data URI.

---

### 3. Recommended Generation Endpoint

The Gradio UI currently calls `generate_image_handler(...)` directly. To support mobile clients, expose an HTTP route (e.g., `POST /magic_quill/generate`) that forwards the same payload to `generate_image_handler`. Although the Python code must not be modified for now, design your mobile API contract using the following schema so it can be wired up later.

**Suggested Request JSON**
```json
{
  "from_frontend": {
    "total_mask": "data:image/png;base64,....",
    "original_image": "data:image/png;base64,....",
    "add_color_image": "data:image/png;base64,....",
    "add_edge_image": "data:image/png;base64,....",
    "remove_edge_image": "data:image/png;base64,...."
  },
  "from_backend": {
    "prompt": "A royal crest with vibrant reds and golds."
  },
  "params": {
    "ckpt_name": "SD1.5/realisticVisionV60B1_v51VAE.safetensors",
    "negative_prompt": "",
    "fine_edge": "disable",
    "grow_size": 15,
    "edge_strength": 0.55,
    "color_strength": 0.55,
    "inpaint_strength": 1.0,
    "seed": -1,
    "steps": 20,
    "cfg": 5.0,
    "sampler_name": "euler_ancestral",
    "scheduler": "karras"
  }
}
```
- `from_frontend` mirrors the object Gradio supplies; all values are base64 PNGs with data URI prefixes.
- `from_backend.prompt` is the positive prompt text supplied by the user.
- `params` groups the slider/radio/dropdown values; `seed = -1` means “random seed” (the handler generates one server-side).

**Suggested Response JSON**
```json
{
  "generated_image": "data:image/png;base64,....",
  "seed": 1234567890,
  "metadata": {
    "ckpt_name": "...",
    "steps": 20,
    "cfg": 5.0
  }
}
```
- `generated_image` is the final PNG in base64.
- Optional metadata helps the client reproduce results or show UI details.

When you later add the FastAPI route, it should:
1. Parse the JSON payload.
2. Call `generate_image_handler` with the same arguments the Gradio button supplies.
3. Return the handler’s output (primarily the base64 image string).

---

### 4. Mobile Client Workflow

1. **Capture Inputs**
   - Let the user select/import the base image.
   - Allow drawing masks/edges and tint overlays locally (same layers currently provided by Gradio).
2. **Prompt Suggestion (optional)**
   - POST to `/magic_quill/guess_prompt` to get hint text.
3. **Preprocess Background (optional)**
   - POST to `/magic_quill/process_background_img` if you need the backend to handle resizing.
4. **Generate Image**
   - POST the payload described in Section 3 to the future `/magic_quill/generate` endpoint.
5. **Display / Save Result**
   - Decode the returned base64 PNG to show in-app or save/share.

All network calls are standard HTTPS POSTs; include authorization headers if you add auth (see below).

---

### 5. Considerations for Production

- **Authentication**: Add an API key or OAuth layer so only trusted clients can call the worker.
- **Rate limiting**: The generation endpoint is GPU-intensive; enforce per-user quotas.
- **Timeouts/retries**: Model inference can take tens of seconds. Configure client and server timeouts accordingly and show progress UI.
- **Chunked uploads**: Very large base64 strings can hit payload limits; consider multipart uploads or presigned URLs if needed.
- **Model assets**: Ensure the server has access to checkpoint files referenced by `ckpt_name` and the same directory layout as the current Gradio environment.
- **Auto-save behavior**: `AUTO_SAVE` currently writes to `output/`. Decide whether remote generations should trigger server-side saves or remain client-only.

---

### 6. Testing Checklist

- Verify `guess_prompt` with minimal payload (only `original_image`) and with all layers present.
- Ensure `process_background_img` respects the server’s `RES` setting (try 256, 512, 1024).
- Once implemented, test the generation route with:
  - Empty negative prompt and random seed.
  - Fixed seed to confirm determinism.
  - Multiple sampler/scheduler combinations.
- Load-test concurrent requests to evaluate GPU memory usage.

This blueprint allows the Kotlin and iOS apps to communicate cleanly with the MagicQuill FastAPI backend while remaining independent of the Gradio interface.

