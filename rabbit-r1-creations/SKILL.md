---
name: rabbit-r1-creations
description: >
  Build Rabbit R1 Creations — tiny 240×282px HTML/CSS/JS apps that run on the
  Rabbit R1 device's built-in WebView. Use this skill whenever the user wants
  to build, design, or modify a Creation for the Rabbit R1. Triggers on phrases
  like "make a Creation", "build an r1 app", "rabbit r1 plugin", "r1 creation",
  "PTT voice app", or any request to write code targeting the Rabbit R1 device.
  Always use this skill even if the user just describes a feature idea without
  explicitly naming it a "Creation" — if it sounds like something that would run
  on an r1, this skill applies. Output is a single index.html (simple cases) or
  a multi-file bundle (index.html + .js + .css) when complexity warrants it.
---

# Rabbit R1 Creations Skill

## Reference material

Full SDK constraints, API details, patterns, and gotchas are in:
`references/r1-creations-sdk.md`

**Read this file at the start of every Creation task.** It contains hard-won
discoveries about silent failures, confirmed working APIs, and native bridge
usage that are not obvious and easy to get wrong.

---

## Workflow

### 1. Understand the idea

Ask the user (if not already clear):
- What does the Creation do? (one sentence)
- Does it need the LLM bridge, microphone, camera, storage, accelerometer?
- Any preference on visual style (dark/light, minimal/rich)?

Keep it brief — don't over-interview. If the concept is obvious, start building.

### 2. Plan before coding

Before writing a line of code, mentally check:
- Which native bridges are needed? (`PluginMessageHandler`, `MediaRecorder`, etc.)
- Are there any known gotchas for those features? (check the SDK doc)
- Will this fit in 240×282px without scrolling?
- Should this be a single `index.html` or split into files?

**Use multi-file layout when:**
- JS logic exceeds ~150 lines
- CSS is substantial and distinct from layout
- The project has clearly separable concerns (e.g., a state machine + UI layer)

Otherwise, a single `index.html` is cleaner and easier to deploy.

### 3. Build the Creation

Follow these rules without exception — they reflect real device constraints:

#### Layout
- **Fixed canvas: 240×282px.** No exceptions.
- `<meta name="viewport" content="width=240, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">`
- `body { width: 240px; height: 282px; overflow: hidden; }`
- No scrolling UI. If content doesn't fit, use internal `overflow` containers (e.g., a scrollable log panel).
- Minimum touch target: **44×44px** for any interactive element.

#### R1-specific API rules
- Always use `addEventListener` — never inline `onclick` in `innerHTML`. They silently fail.
- Touch: attach `touchstart` to `document.body`, check `e.target`, always call `e.preventDefault()`.
- Camera: use `facingMode: 'environment'` for rear, `'user'` for front.
- Mic: call `getUserMedia({ audio: true })` on page load to pre-warm (triggers permission prompt early).
- Vision: capture at **160×120 @ JPEG 0.5 quality** (~8–12KB). Prepend `IMAGE_DATA_URI:` label in the message.
- LLM bridge payloads: keep under **~15KB** total to avoid `No LLM Response Available`.
- Storage: always base64-encode JSON values with `btoa(JSON.stringify(...))` before saving.

#### What NOT to use
- No WebGL (Flutter WebView doesn't support it — use Canvas 2D only)
- No `webkitSpeechRecognition` / `SpeechRecognition` (blocked on device)
- No audio base64 in LLM bridge messages (returns error)
- No image generation via LLM bridge (returns `CANNOT_GENERATE`)
- No HTTP for `getUserMedia` (HTTPS / `*.rabbitos.app` only)

#### Code quality
- Add comments for every non-obvious section, especially bridge calls and event wiring.
- Use `const isR1 = typeof PluginMessageHandler !== 'undefined'` to guard bridge calls — makes the Creation testable in a desktop browser too.
- Structure code clearly: init → event wiring → helpers → LLM handler.
- Prefer named functions over anonymous lambdas for readability.

#### Visual design
- Dark background (`#000` or very dark) works best on the r1's screen.
- Use `font-family: monospace` or a clean sans-serif at small sizes (10–13px).
- Provide clear visual feedback for state: recording, thinking, ready (e.g., border color changes via `updateAppBorderColor(hex)`).
- Keep UI elements minimal — the screen is tiny. Prioritize one primary action per screen.

### 4. Deliver the output

- For a **single-file** Creation: produce `index.html` directly.
- For a **multi-file** Creation: produce `index.html`, `app.js`, `style.css` (and any others needed), and briefly explain the structure.
- Always include a short comment block at the top of each file explaining what it does.
- Remind the user to deploy via [rabbit.tech/creations](https://rabbit.tech/creations).

---

## Common patterns (quick reference)

Detailed, copy-paste-ready implementations of these patterns are in the SDK doc:

- **PTT Voice Assistant** — `longPressStart` → `MediaRecorder` → Whisper STT → LLM bridge
- **Vision / Camera** — `getUserMedia` → Canvas frame capture → embed base64 in LLM message
- **Structured LLM responses** — prompt for JSON → parse `data.data || data.message`
- **Persistent storage** — `creationStorage.plain.setItem/getItem` with btoa/atob
- **Accelerometer** — `window.creationSensors.accelerometer.start(cb, { frequency: 60 })`
- **Hardware events** — `sideClick`, `longPressStart`, `longPressEnd`, `scrollUp`, `scrollDown`

---

## Minimal boilerplate

The SDK doc contains a full minimal boilerplate. Use it as the starting point for
every Creation — it includes viewport meta, body sizing, `isR1` guard, hardware
event stubs, and the `askLLM` helper.
