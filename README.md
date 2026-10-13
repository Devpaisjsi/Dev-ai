# Dev AI v1

A small, dark, Hindi/English browser AI assistant built with HTML, CSS, JavaScript and Hugging Face Transformers.js.

## Files
- `index.html` — chat interface and model inference
- `manifest.json` — PWA metadata
- `sw.js` — caches the app shell for repeat visits (model files are not cached by this worker)
- `icon.svg` — app icon

## Model
Default: `onnx-community/Qwen3-0.6B-Instruct-ONNX`, loaded through Transformers.js. It tries WebGPU with 4-bit float16 weights when available, then falls back to CPU/WASM with 4-bit quantization. This is a compact model, not ChatGPT-level. Its model files can be large (potentially hundreds of MB or more), so use Wi-Fi for the first load and check free storage.

## Run
Serve these files over HTTPS or localhost. Opening `index.html` directly as a `file://` URL will not work reliably because ES modules and service workers require an appropriate origin.

## Publish with GitHub Pages
1. Create a **public** GitHub repository named `dev-ai`.
2. Upload all four app files (`index.html`, `manifest.json`, `sw.js`, `icon.svg`) to the repository root.
3. Open repository **Settings → Pages**.
4. Under **Build and deployment**, choose **Deploy from a branch**, select `main` and `/ (root)`, then Save.
5. Wait for the published URL to appear in Settings → Pages.
6. Open the HTTPS URL in Chrome on Android, wait for the model download, then test.

## Install as an Android PWA
Open the published site in Chrome → ⋮ → **Install app** or **Add to Home screen**. Exact wording varies by browser.

## Limitations
- The model is intentionally small; it is not as capable as ChatGPT and may be weak in Hindi.
- First model download needs internet and may consume substantial mobile data and storage.
- Speed depends on device memory, browser, and supported acceleration.
- The service worker caches the app shell only; it does not guarantee offline model inference.
- GitHub Pages is static hosting. Do not put private API keys or secrets in this repository.
- Before commercial use or redistribution, check the model and dependency licences.
