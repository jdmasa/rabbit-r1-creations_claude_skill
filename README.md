# rabbit-r1-creations skill

A Claude skill for building **Rabbit R1 Creations** — tiny HTML/CSS/JS apps that run natively on the Rabbit R1 device.

---

## What is a Creation?

A Creation is a static web app (HTML + CSS + JS) that runs inside the Rabbit R1's built-in WebView browser. You build it like a regular webpage, but it's designed exclusively for the r1's fixed 240×282px screen and has access to native device features through JavaScript bridge APIs injected by rabbitOS — things like the LLM, microphone, camera, accelerometer, and persistent storage.

Creations are deployed using a QR Code and run entirely client-side. No server code runs on the device.

---

## What this skill does

When this skill is installed, Claude gains deep knowledge of the R1's WebView constraints, native bridge APIs, and common silent failure modes. Instead of guessing at what works, Claude follows a set of hard-won rules from real device testing, so the code it produces is actually correct on the first try.

Specifically, Claude will:

- Design exclusively for the **240×282px fixed canvas** with no scrolling UI
- Use the correct **native bridge APIs** (`PluginMessageHandler`, `MediaRecorder`, hardware events, etc.)
- Avoid known **silent failure traps** (e.g. `onclick` in `innerHTML`, `webkitSpeechRecognition`, WebGL, audio base64 in LLM messages)
- Write clean, well-commented code structured for readability and easy modification
- Output a **single `index.html`** for simple apps, or a **multi-file bundle** (`index.html` + `app.js` + `style.css`) when complexity warrants it
- Provide visual state feedback (border color changes, status text) so the app feels responsive on-device

---

## How to use it

Just describe what you want your Creation to do in plain language. Claude will ask a few clarifying questions if needed (LLM, mic, camera?) and then build it.

**Example prompts that trigger this skill:**

- *"Make a Creation that lets me ask questions by holding the side button"*
- *"Build an r1 app that identifies objects using the camera"*
- *"I want a Creation that tracks my daily mood and saves entries"*
- *"Create a voice-controlled timer for the Rabbit R1"*
- *"Make a PTT app that translates what I say into Spanish"*

You don't need to say "Creation" explicitly — if it sounds like something that would run on an r1, the skill kicks in.

---

## What the skill knows

The skill bundles `references/r1-creations-sdk.md`, a comprehensive reference covering:

| Topic | Details |
|---|---|
| Screen & layout | 240×282px fixed canvas, viewport meta, overflow rules |
| WebView constraints | What silently fails and why |
| Confirmed working APIs | `getUserMedia`, `MediaRecorder`, `AudioContext`, Canvas 2D, `fetch`, WebSocket, WebRTC |
| Native bridges | LLM (`PluginMessageHandler`), storage, accelerometer, touch, button events |
| LLM bridge | How to send prompts, receive responses, handle structured JSON, payload limits |
| Vision (undocumented) | How to embed camera frames in LLM messages for multimodal queries |
| Microphone & STT | Recording with `MediaRecorder`, free Whisper endpoint for transcription |
| Hardware events | PTT button (`longPressStart`/`longPressEnd`), scroll wheel, side click |
| Persistent storage | `creationStorage` API with base64 encoding pattern |
| Accelerometer | Tilt + raw values at up to 100Hz |
| Things that don't work | WebGL, `SpeechRecognition`, image generation, audio via LLM bridge |

---

## Output format

| App complexity | Output |
|---|---|
| Simple (single interaction, <150 lines JS) | Single `index.html` |
| Complex (state machines, multiple views, heavy logic) | `index.html` + `app.js` + `style.css` |

Every output includes a comment block at the top explaining what the file does, and uses `const isR1 = typeof PluginMessageHandler !== 'undefined'` so you can also open and test it in a regular desktop browser.

---

## Deployment

Once Claude gives you the files, deploy using the QR Code. Host your `index.html` (and any companion files) on a server, give your Creation a name, and it'll be available on your r1.

---

## Skill contents

```
rabbit-r1-creations/
├── SKILL.md                        # Instructions Claude follows when building Creations
├── README.md                       # This file
└── references/
    └── r1-creations-sdk.md         # Full SDK reference, API details, patterns & gotchas
```
