# Design Document — button-click-sound

## Overview

This feature adds a click-triggered audio interaction to the "Visit My Portfolio →" button in `index.html`. When the button is clicked, a short sound clip plays and the browser simultaneously opens `https://ahmadammar.my.id` in a new tab. The entire implementation lives inside the existing `index.html` — no external files, no build tools, no dependencies.

The design is intentionally minimal: a hidden `<audio>` element carries the sound asset as a Base64-encoded data URI, and a small inline `<script>` block intercepts the button's click event to play the audio and open the target URL.

### Key design goals

- Navigation is never delayed or blocked by audio
- Audio failures are swallowed silently — the button always navigates
- Autoplay policy is satisfied by calling `.play()` inside the synchronous click handler
- The page remains a single static HTML file deployable from the file system

---

## Architecture

The implementation uses three cooperating pieces that already exist or will be added to `index.html`:

```
User click
    │
    ▼
click event handler (inline <script>)
    ├── audio.currentTime = 0
    ├── audio.play()  ──→ Promise  ──→ .catch() swallows errors
    └── window.open(url, '_blank')  ──→  navigates immediately
```

**No framework, no module system, no external requests.** The handler fires in the same microtask as the user gesture, which satisfies all modern browsers' autoplay policies.

### Sequencing rationale

`window.open()` is called **after** `audio.play()` is called but **without awaiting** the play promise. This means:

1. `.play()` is invoked synchronously inside the user-gesture handler → autoplay policy satisfied.
2. `window.open()` executes in the same synchronous pass → navigation is immediate (≤ 1 second per Requirement 1.2).
3. The promise rejection handler (`.catch()`) runs asynchronously only if playback fails → silent fallback, navigation already dispatched.

---

## Components and Interfaces

### 1. `<audio>` element

```html
<audio id="btn-sound" preload="auto">
  <source src="data:audio/mpeg;base64,..." type="audio/mpeg" />
</audio>
```

| Attribute | Value | Reason |
|---|---|---|
| `id` | `btn-sound` | Allows the script to reference it without querying by class |
| `preload` | `auto` | Browser loads the audio on page load; reduces first-click latency |
| `src` | Base64 data URI | Self-contained; no network request; satisfies Requirement 5.2 |

The element is hidden by default (`<audio>` elements render nothing). No `autoplay` or `loop` attributes are set.

### 2. Click event handler (`<script>` block)

Located at the bottom of `<body>`, after all DOM elements are present:

```js
(function () {
  var btn  = document.querySelector('.btn');
  var clip = document.getElementById('btn-sound');

  if (!btn || !clip) return;   // defensive: no-op if DOM is unexpected

  btn.addEventListener('click', function (e) {
    // Reset so re-clicks always play from the start (Requirement 1.1)
    clip.currentTime = 0;

    // Play inside user-gesture handler to satisfy Autoplay Policy (Requirement 4.1)
    // Swallow all errors silently — navigation must not be blocked (Requirements 1.4, 2.2, 2.3)
    clip.play().catch(function (err) {
      // Log load failures for debugging (Requirement 2.1)
      // Suppress autoplay/NotAllowedError silently (Requirement 2.2)
      if (err.name !== 'NotAllowedError') {
        console.error('[btn-sound] playback error:', err);
      }
    });

    // Navigation proceeds regardless of audio outcome (Requirements 1.2, 1.3)
    // href + target are preserved on the <a> element (Requirement 3.1);
    // window.open mirrors what the browser would do for target="_blank"
    window.open(btn.href, '_blank', 'noopener,noreferrer');

    // Prevent the default anchor navigation to avoid a duplicate tab
    e.preventDefault();
  });
}());
```

**Why `e.preventDefault()` + `window.open()`?**  
Without it, the browser fires both the default anchor navigation and `window.open()`, opening two tabs. Calling `preventDefault()` and manually opening the tab gives us full control over timing while preserving the visual/semantic CTA anchor element (Requirement 3.1).

**Why an IIFE?**  
Keeps all variables scoped; avoids polluting the global namespace of a page that might later gain more scripts.

### 3. Sound asset

The `<audio src>` value is a Base64-encoded data URI of the chosen `.mp3` file. The encoding is done once, offline, and pasted into the HTML. This means:

- Zero network round-trips for the audio asset
- Works from `file://` URLs (Requirement 5.4)
- Satisfies the single-file constraint (Requirement 5.3)

> **Implementation note**: The actual Base64 string is not included in this design document. During implementation, the chosen audio clip should be encoded with a tool such as `base64 -i clip.mp3` (macOS/Linux) or `certutil -encode clip.mp3` (Windows) and inserted as `data:audio/mpeg;base64,<encoded-string>`.

---

## Data Models

This feature introduces no persistent data structures, no API calls, and no state beyond what the browser's `HTMLAudioElement` already manages.

The only runtime state is the `HTMLAudioElement` object (`clip`), which holds:

| Property | Type | Usage |
|---|---|---|
| `currentTime` | `number` (seconds) | Reset to `0` on each click to replay from start |
| `paused` | `boolean` | Read-only; not used directly but relevant for debugging |
| `error` | `MediaError \| null` | Populated by the browser on load failure; surfaced via the `'error'` event |

No custom data model is needed.

---

## Correctness Properties

### Property 1: Audio Always Resets Before Replay

For every click event, `clip.currentTime` is set to `0` before `clip.play()` is called. Repeated clicks always replay the sound from the beginning, regardless of playback state.

**Validates: Requirements 1.1**

### Property 2: Navigation Is Never Blocked by Audio

For every click event, `window.open()` is called in the same synchronous execution frame as `clip.play()`, without awaiting the play promise. Navigation completes independently of the audio outcome.

**Validates: Requirements 1.2, 1.3, 1.4**

### Property 3: Audio Failures Are Swallowed Silently

For every `play()` rejection, navigation has already been dispatched before the rejection handler executes. A `NotAllowedError` must never produce a `console.error`; any other rejection must log to `console.error` but must not affect navigation.

**Validates: Requirements 2.1, 2.2, 2.3**

### Property 4: Button Anchor Semantics Are Preserved

At all times, the `.btn` element retains its `href`, `target="_blank"`, and `rel="noopener noreferrer"` attributes. The click handler augments the button's behaviour and never removes or overrides these attributes.

**Validates: Requirements 3.1, 3.2**

### Property 5: Self-Contained Single File

At all times, the audio asset is embedded as a Base64 data URI (`data:audio/mpeg;base64,...`). No external network request is made for the audio clip, and the page is deployable as a single `.html` file from a `file://` URL.

**Validates: Requirements 5.1, 5.2, 5.3, 5.4**

---

## Error Handling

| Failure scenario | Browser behaviour | Handler response |
|---|---|---|
| Audio file not found / network error | `audio.error` set; `play()` rejects | `.catch()` logs to console; navigation already dispatched |
| `NotAllowedError` (autoplay blocked) | `play()` rejects with `DOMException` | `.catch()` checks `err.name`; swallowed silently per Requirement 2.2; navigation already dispatched |
| Any other `play()` rejection | `play()` rejects | `.catch()` logs to console; navigation already dispatched |
| `btn` or `clip` element missing | — | Guard clause `if (!btn \|\| !clip) return` prevents any runtime error; anchor default behaviour is unaffected |
| `window.open()` blocked by popup blocker | Browser may swallow the open | The `<a href>` is preserved as a fallback; the user can click again or use the native anchor |

### Requirement 4.3 conflict note

Requirement 4.3 asks for a **visible** user-facing error when autoplay is blocked. Requirement 2.2 asks to **suppress** any visible indication. These are contradictory. The design follows Requirement 2.2 (silent suppression) because:

1. Autoplay blocks are common on first visit and represent a normal browser state, not a user-visible error.
2. A visible error would degrade UX for users on browsers with strict autoplay policies.
3. Navigation proceeds regardless, so the user is never stuck.

If Requirement 4.3 is intentionally kept, a non-intrusive toast notification (CSS-only, auto-dismissing) could be added in a future iteration without affecting the navigation flow.

---

## Testing Strategy

Property-based testing is **not applicable** to this feature. The logic is entirely event-driven and side-effect-based (`window.open`, `HTMLAudioElement.play`, DOM mutation). There are no pure functions with a wide input space where randomised input variation would uncover additional bugs. All testable acceptance criteria are specific scenarios or structural checks.

### Unit tests (example-based)

Each test uses `jsdom` (or equivalent) with `HTMLAudioElement` and `window.open` mocked.

| Test ID | Requirement | Scenario | Assertion |
|---|---|---|---|
| U-01 | 1.1 | Click when audio not playing | `audio.currentTime === 0`, `play()` called |
| U-02 | 1.1 | Click when audio already playing (`currentTime > 0`) | `audio.currentTime` reset to `0`, `play()` called |
| U-03 | 1.2 | Single click | `window.open` called with `https://ahmadammar.my.id` and `'_blank'` |
| U-04 | 1.3 | Single click | `window.open` called before audio promise resolves |
| U-05 | 1.4 | `play()` rejects | `window.open` still called; no unhandled rejection |
| U-06 | 2.1 | Audio `'error'` event fires | `console.error` called with message; `window.open` called |
| U-07 | 2.2 | `play()` rejects with `NotAllowedError` | No `console.error` called; `window.open` called |
| U-08 | 2.3 | `play()` rejects with arbitrary error | `window.open` called |
| U-09 | 3.1 | Inspect modified `index.html` DOM | Button has `href`, `target="_blank"`, `rel="noopener noreferrer"` |
| U-10 | 4.1 | Inspect call order in click handler | `play()` called synchronously before any `await` or `setTimeout` |

### Smoke / structural checks

| Check | Requirement | Method |
|---|---|---|
| S-01 | 5.1 | No `<script src="...">` tags pointing to external URLs in `index.html` |
| S-02 | 5.2 | `<audio src>` value starts with `data:audio/` (Base64 URI) |
| S-03 | 5.3 | Repository contains exactly one `.html` file after implementation |
| S-04 | 5.4 | Open `index.html` via `file://` in browser; verify button navigates and audio plays |

### Manual / integration checks

| Check | Requirement | Method |
|---|---|---|
| M-01 | 4.2 | Click button, verify audio starts within 200 ms (subjective listen + DevTools) |
| M-02 | 2.4 | Click button, measure time to interactive on `ahmadammar.my.id` |
| M-03 | 3.2 | Visual diff of button styles before and after implementation |
| M-04 | 3.3 | Click button, confirm no spinner or cursor change within 100 ms |
