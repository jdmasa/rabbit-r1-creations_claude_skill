# Rabbit R1 — Creations Plugin Development Guide

> Hard-won discoveries from hands-on experimentation + official SDK source code.
> Drop this file into any new conversation to skip the discovery phase.

**SDK repo:** https://github.com/rabbit-hmi-oss/creations-sdk
**Raw files accessible at:** `https://raw.githubusercontent.com/rabbit-hmi-oss/creations-sdk/refs/heads/main/plugin-demo/...`

---

## What is a Creation?

A Creation is a static HTML/CSS/JavaScript website that runs inside the r1's built-in WebView browser. Deployed via [rabbit.tech/creations](http://rabbit.tech/creations) and rendered on-device. No server-side code runs on the r1 — all logic is client-side JS with optional calls to external APIs.

---

## Screen & Layout

- **Fixed size: 240×282 pixels.** Design exclusively for this size.
- No scrolling UI — everything must fit or use internal `overflow` containers.
- Use `<meta name="viewport" content="width=240, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">`.
- Set explicit `width: 240px; height: 282px; overflow: hidden` on `body`.
- Touch-friendly minimum button size: **44×44px**.

---

## WebView Constraints (Critical — Silent Failures)

| Constraint | Detail |
|---|---|
| **HTTPS mandatory** | `navigator.mediaDevices` is `undefined` over HTTP. Creations on `*.rabbitos.app` are whitelisted. |
| **No WebGL** | Flutter-based WebView has no WebGL. Use Canvas 2D only. |
| **`onclick` in dynamic HTML** | `onclick` attributes in `innerHTML` do NOT fire. Always use `addEventListener`. |
| **Touch events** | `touchstart` on specific elements may not fire. Attach to `document.body` and check `e.target`. Always call `e.preventDefault()` to avoid double-firing with click. |
| **No native image generation** | Magic Camera is OS-level, not accessible to Creations. |
| **Audio via LLM bridge** | Sending base64 audio in the message string returns `No LLM Response Available`. Not supported. |

---

## Available Web APIs (Confirmed)

| API | Status | Notes |
|---|---|---|
| `navigator.mediaDevices.getUserMedia` | ✅ | HTTPS only. Works for `video` and `audio`. |
| `MediaRecorder` | ✅ | MIME types: `audio/webm`, `audio/webm;codecs=opus` |
| `AudioContext` | ✅ | |
| `Canvas 2D` | ✅ | Only rendering option — no WebGL |
| `WebSocket` | ✅ | |
| `RTCPeerConnection` (WebRTC) | ✅ | |
| `fetch` | ✅ | Can call any external API |
| `FileReader` | ✅ | Blob → base64 conversion |
| `webkitSpeechRecognition` | ⚠️ | Exists but immediately aborts — blocked from Google speech servers in r1 sandbox. Not usable. |
| `SpeechRecognition` | ❌ | Not available |
| `WebGL` | ❌ | Not available |

---

## Native Bridges (JavaScript Globals)

Injected by rabbitOS into every Creation's WebView:

```javascript
PluginMessageHandler.postMessage(payload)    // LLM + TTS
FlutterButtonHandler.postMessage(payload)    // button events
AccelerometerHandler.postMessage(payload)    // accelerometer
CreationStorageHandler.postMessage(payload)  // persistent storage
TouchEventHandler.postMessage(payload)       // touch events
closeWebView                                 // close the creation
```

---

## LLM Bridge — `PluginMessageHandler`

### Sending

```javascript
PluginMessageHandler.postMessage(JSON.stringify({
  message: "your prompt",   // required — the text prompt
  useLLM: true,             // required
  wantsR1Response: false,   // optional — true = speak via r1 speaker
  wantsJournalEntry: false  // optional — true = save response to r1 journal
}));
```

### Receiving

```javascript
window.onPluginMessage = function(data) {
  // data.message = plain text response
  // data.data    = JSON string (for structured responses)
  const text = data.message || data.data || JSON.stringify(data);
};
```

### Handling structured JSON responses

Ask the LLM to return JSON and parse `data.data` or `data.message`:

```javascript
// Request
PluginMessageHandler.postMessage(JSON.stringify({
  message: 'Return ONLY valid JSON: {"result": "value"}',
  useLLM: true
}));

// Handle
window.onPluginMessage = function(data) {
  const source = data.data || data.message;
  try {
    const parsed = typeof source === 'string' ? JSON.parse(source) : source;
    // use parsed.result etc.
  } catch(e) {
    // plain text fallback
  }
};
```

### Known limitations
- Text in, text out only (officially).
- No streaming, no tool calling, no MCP.
- Payload size limit: keep messages under **~15KB** or get `No LLM Response Available`.
- Audio base64 in message → `No LLM Response Available`.
- Image generation → `CANNOT_GENERATE` (text-only output).

---

## Vision (Confirmed Working ✅ — Undocumented)

The r1's underlying LLM is **multimodal** and processes base64 JPEG data URIs embedded in the message string.

```javascript
// 1. Start camera
const stream = await navigator.mediaDevices.getUserMedia({
  video: { facingMode: 'environment' }, audio: false
});
const video = document.getElementById('video');
video.srcObject = stream;

// 2. Capture frame (keep small — ~8-12KB)
const canvas = document.createElement('canvas');
canvas.width = 160; canvas.height = 120;
canvas.getContext('2d').drawImage(video, 0, 0, 160, 120);
const base64 = canvas.toDataURL('image/jpeg', 0.5);

// 3. Send with embedded image
PluginMessageHandler.postMessage(JSON.stringify({
  message: 'Describe what you see in this image.\nIMAGE_DATA_URI: ' + base64,
  useLLM: true,
  wantsR1Response: false
}));
```

**Tips:**
- Capture at **160×120 @ 0.5 JPEG quality** — keeps payload ~8–12KB.
- The `IMAGE_DATA_URI:` label prefix helps the model locate the data.
- Model returns text description only — cannot return or generate images.

---

## Image Generation — `com.r1.pixelart` Bridge (Confirmed Working ✅ — Undocumented)

The r1 exposes an **image generation bridge** via `PluginMessageHandler` using a special `pluginId`. This is how the official "AI Camera Styles" Creation works. Instead of `useLLM: true`, you send `pluginId: "com.r1.pixelart"` along with an `imageBase64` field and a text prompt describing the transformation.

The r1 takes your input image + prompt and returns a **generated/transformed image** back via `window.onPluginMessage`.

### Sending

```javascript
// 1. Capture from canvas or video element
const canvas = document.createElement('canvas');
canvas.width  = video.videoWidth;   // capture at full video resolution
canvas.height = video.videoHeight;
canvas.getContext('2d').drawImage(video, 0, 0);

// 2. Get full data URI (including the data:image/jpeg;base64, prefix)
const imageDataUri = canvas.toDataURL('image/jpeg', 0.8);

// 3. Send — note: imageBase64 takes the FULL data URI, not just the base64 part
PluginMessageHandler.postMessage(JSON.stringify({
  message:     "Take a picture in the style of a Simpsons cartoon.",  // image transformation prompt
  pluginId:    "com.r1.pixelart",   // required — activates image generation mode
  imageBase64: imageDataUri         // full data URI string
}));
```

### Receiving

The response comes back via `window.onPluginMessage` with status events:

```javascript
window.onPluginMessage = function(data) {
  if (data.status === 'processing') {
    // AI is working — show a loading state
    statusEl.textContent = 'AI is processing your image...';
  } else if (data.status === 'complete') {
    // Generation done — the transformed image is delivered separately
    // (likely applied directly to the device UI / Magic Camera output)
    statusEl.textContent = 'AI transformation complete!';
  } else if (data.error) {
    statusEl.textContent = 'Error: ' + data.error;
  }
};
```

### Helper function (from official SDK wrapper)

```javascript
function captureAndTransform(videoOrCanvas, prompt) {
  let canvas;
  if (videoOrCanvas instanceof HTMLVideoElement) {
    canvas = document.createElement('canvas');
    canvas.width  = videoOrCanvas.videoWidth;
    canvas.height = videoOrCanvas.videoHeight;
    canvas.getContext('2d').drawImage(videoOrCanvas, 0, 0);
  } else {
    canvas = videoOrCanvas; // already a canvas
  }

  // Split off the base64 data without the prefix
  const base64Only = canvas.toDataURL('image/jpeg', 0.8).split(',')[1];

  PluginMessageHandler.postMessage(JSON.stringify({
    message:     prompt,
    pluginId:    'com.r1.pixelart',
    imageBase64: 'data:image/jpeg;base64,' + base64Only  // reconstruct full URI
  }));
}
```

### Example transformation prompts (from official app source)

These are the exact prompts Rabbit uses in their AI Camera Styles Creation:

```
"Take a picture in the style of a Simpsons cartoon."
"Take a picture in the style of Pixar Animation."
"Take a picture and make it LEGO minifigures."
"Take a picture in the style of Studio Ghibli anime style-soft colors, whimsical atmosphere, and hand-drawn aesthetic."
"Take a picture in the style of Minecraft."
"Take a picture and make the subject a superhero in a comic book."
"Take a picture in the style of a Norman Rockwell painting."
"Take a picture in the style of a Watercolor painting."
"Take a picture and turn the subject into a Funko Pop."
"Take a picture in the style of a vintage movie poster. Come up with the film's title, director, and actor names based on the image."
"Take a picture and place the subject in a scene from the movie Star Wars."
"Take a picture and make it picture perfect - improve lighting, colors, and overall composition. Make it photorealistic. 8k resolution."
```

The prompt pattern is always: **action verb** (`Take a picture`) + **transformation description**. The model understands "the subject" to mean the person/object in the image.

### Key differences vs standard LLM bridge

| Aspect | Standard LLM (`useLLM: true`) | Image gen (`pluginId: "com.r1.pixelart"`) |
|---|---|---|
| Input | Text prompt (+ optional embedded image) | Text prompt + `imageBase64` field |
| Output | Text response in `data.message` | Status events (`processing` / `complete`); image delivered via device |
| Image size | Keep small (~8–12KB) to stay under 15KB limit | Full camera resolution OK (1280×720 @ 0.8 JPEG seen in official app) |
| Response handler | `data.message` or `data.data` | `data.status === 'complete'` |

### Also works as a status ping

You can send a message with just `pluginId` and no image to initialize/check the bridge:

```javascript
// Used in official app to signal readiness
PluginMessageHandler.postMessage(JSON.stringify({
  message:  'AI Camera Styles initialized',
  pluginId: 'com.r1.pixelart'
}));
```

---

## Camera

```javascript
// Rear camera (r1 label: "camera2 0, facing back")
const stream = await navigator.mediaDevices.getUserMedia({
  video: { facingMode: 'environment' }, audio: false
});

// Front camera
const stream = await navigator.mediaDevices.getUserMedia({
  video: { facingMode: 'user' }, audio: false
});

video.srcObject = stream;

// Always clean up
stream.getTracks().forEach(t => t.stop());
```

---

## Microphone & STT

### Pre-warm mic (do this on page load)

Always call `getUserMedia` on load to trigger the permission prompt before recording:

```javascript
async function warmMic() {
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  stream.getTracks().forEach(t => t.stop());
}
```

### Recording

```javascript
const micStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
const recorder  = new MediaRecorder(micStream, { mimeType: 'audio/webm;codecs=opus' });
const chunks    = [];

recorder.ondataavailable = e => { if (e.data.size > 0) chunks.push(e.data); };
recorder.onstop = () => {
  const blob = new Blob(chunks, { type: 'audio/webm;codecs=opus' });
  // ~6KB/sec — send to Whisper
};

recorder.start(100); // collect every 100ms
// ...
recorder.stop();
```

### STT via Whisper (free, no API key needed)

```javascript
const formData = new FormData();
formData.append('audio_file', blob, 'audio.webm');

const res = await fetch(
  'https://masatrad-whisper.hf.space/asr?output=txt&language=en&encode=true',
  { method: 'POST', body: formData }
);
const transcript = (await res.text()).trim();
```

**Endpoint details** (`https://masatrad-whisper.hf.space`):
- `POST /asr` — transcribe audio
- `POST /detect-language` — detect spoken language
- Query params: `output` (txt/json/srt/vtt), `language`, `task` (transcribe/translate), `encode`
- Body: `multipart/form-data` with `audio_file` field

---

## Hardware Events

```javascript
window.addEventListener('sideClick',      handler); // PTT tap
window.addEventListener('longPressStart', handler); // PTT hold start
window.addEventListener('longPressEnd',   handler); // PTT hold end
window.addEventListener('scrollUp',       handler); // scroll wheel up
window.addEventListener('scrollDown',     handler); // scroll wheel down
```

---

## Accelerometer

```javascript
// Check availability first
if (window.creationSensors?.accelerometer?.isAvailable) {
  const available = await window.creationSensors.accelerometer.isAvailable();
  if (!available) return;
}

// Start
window.creationSensors.accelerometer.start((data) => {
  // Tilt values (normalized -1.0 to 1.0)
  data.tiltX  // left/right
  data.tiltY  // forward/back
  data.tiltZ  // face up/down

  // Raw values (m/s²)
  data.rawX
  data.rawY
  data.rawZ
}, { frequency: 60 }); // 10, 30, 60, or 100 Hz

// Stop
window.creationSensors.accelerometer.stop();
```

---

## Persistent Storage

```javascript
// Save (always base64-encode JSON to be safe)
await window.creationStorage.plain.setItem(
  'my_key',
  btoa(JSON.stringify(myData))
);

// Load
const raw  = await window.creationStorage.plain.getItem('my_key');
const data = JSON.parse(atob(raw));
```

---

## Dynamic Border Color

The app border color can be changed at runtime — useful for visual feedback:

```javascript
function updateAppBorderColor(hexColor) {
  document.getElementById('app').style.borderColor = hexColor;
}
// e.g. updateAppBorderColor('#ff0000');
```

---

## Things That Do NOT Work

| Feature | Result |
|---|---|
| Image generation via standard LLM bridge (`useLLM: true`) | Returns `CANNOT_GENERATE` — use the pixelart bridge instead (see below) |
| Audio base64 via LLM bridge | Returns `No LLM Response Available` |
| `webkitSpeechRecognition` | Aborts immediately — network-sandboxed |
| `SpeechRecognition` | Not available |
| WebGL | Not supported in Flutter WebView |
| `getUserMedia` over HTTP | Requires HTTPS / secure context |
| Magic Camera API | OS-level, not exposed to Creations |
| MCP / tool calling via bridge | Not supported |

---

## Full PTT Voice Assistant Pattern

```javascript
let micStream = null, recorder = null, chunks = [], recording = false;

// 1. Init on load
async function init() {
  micStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
}

// 2. PTT hold → record
window.addEventListener('longPressStart', () => {
  if (recording) return;
  recording = true;
  chunks = [];
  recorder = new MediaRecorder(micStream, { mimeType: 'audio/webm;codecs=opus' });
  recorder.ondataavailable = e => { if (e.data.size > 0) chunks.push(e.data); };
  recorder.onstop = onStop;
  recorder.start(100);
});

// 3. Release → stop
window.addEventListener('longPressEnd', () => {
  if (!recording) return;
  recording = false;
  recorder.stop();
});

// 4. Whisper → LLM
async function onStop() {
  const blob = new Blob(chunks, { type: 'audio/webm;codecs=opus' });
  const fd   = new FormData();
  fd.append('audio_file', blob, 'audio.webm');

  const res        = await fetch('https://masatrad-whisper.hf.space/asr?output=txt&language=en&encode=true', { method: 'POST', body: fd });
  const transcript = (await res.text()).trim();

  window.onPluginMessage = data => {
    const reply = data.message || data.data || JSON.stringify(data);
    // display reply
  };

  PluginMessageHandler.postMessage(JSON.stringify({
    message: transcript,
    useLLM: true,
    wantsR1Response: false
  }));
}

init();
```

---

## Full Vision Pattern

```javascript
// 1. Start camera
const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
document.getElementById('video').srcObject = stream;

// 2. PTT tap → capture → describe
window.addEventListener('sideClick', () => {
  const canvas = document.createElement('canvas');
  canvas.width = 160; canvas.height = 120;
  canvas.getContext('2d').drawImage(document.getElementById('video'), 0, 0, 160, 120);
  const base64 = canvas.toDataURL('image/jpeg', 0.5);

  window.onPluginMessage = data => {
    const desc = data.message || data.data;
    // display description
  };

  PluginMessageHandler.postMessage(JSON.stringify({
    message: 'Describe what you see.\nIMAGE_DATA_URI: ' + base64,
    useLLM: true,
    wantsR1Response: false
  }));
});
```

---

## Minimal Boilerplate

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=240, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>My Creation</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      width: 240px;
      height: 282px;
      overflow: hidden;
      background: #000;
      color: #fff;
      font-family: monospace;
      font-size: 11px;
    }
    #app {
      width: 240px;
      height: 282px;
      border: 5px solid #333; /* dynamic color target */
      display: flex;
      flex-direction: column;
    }
  </style>
</head>
<body>
<div id="app">
  <!-- UI here -->
</div>
<script>
  // Detect r1 vs browser
  const isR1 = typeof PluginMessageHandler !== 'undefined';

  // Hardware events
  window.addEventListener('sideClick',      () => {});
  window.addEventListener('longPressStart', () => {});
  window.addEventListener('longPressEnd',   () => {});
  window.addEventListener('scrollUp',       () => {});
  window.addEventListener('scrollDown',     () => {});

  // LLM helper
  function askLLM(prompt, onResponse) {
    window.onPluginMessage = function(data) {
      const reply = data.message || data.data || JSON.stringify(data);
      onResponse(reply, data);
    };
    if (isR1) {
      PluginMessageHandler.postMessage(JSON.stringify({
        message: prompt,
        useLLM: true,
        wantsR1Response: false
      }));
    }
  }
</script>
</body>
</html>
```
