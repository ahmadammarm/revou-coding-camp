# Implementation Plan: button-click-sound

## Overview

All changes live inside the single existing `index.html` file. The plan is:

1. Encode the audio asset as a Base64 data URI and embed it as a hidden `<audio>` element.
2. Add an inline `<script>` block at the bottom of `<body>` that intercepts the button's click event, resets and plays the audio, and manually opens the target URL — then prevents the default anchor navigation to avoid a duplicate tab.
3. Verify the implementation with unit tests using jsdom and structural smoke checks.

No external files, build tools, or dependencies are introduced.

---

## Tasks

- [x] 1. Prepare the audio asset

  - [x] 1.1 Select and encode the sound clip as a Base64 data URI
    - Choose a short audio clip (`.mp3`, ≤ 5 seconds recommended).
    - Encode it offline: `base64 -i clip.mp3` (macOS/Linux) or `certutil -encode clip.mp3 out.txt` (Windows).
    - Keep the resulting Base64 string ready for insertion in the next task.
    - _Requirements: 5.2_

- [x] 2. Embed the `<audio>` element in `index.html`

  - [x] 2.1 Add a hidden `<audio id="btn-sound">` element to the `<body>`
    - Insert immediately before the closing `</body>` tag.
    - Set `preload="auto"` so the clip is loaded on page load, minimising first-click latency.
    - Set the `<source src>` to the Base64 data URI from task 1.1 with `type="audio/mpeg"`.
    - Do not set `autoplay` or `loop` attributes.
    - _Requirements: 1.1, 4.2, 5.1, 5.2, 5.3_

- [ ] 3. Add the click-event handler script

  - [ ] 3.1 Add an inline `<script>` block (IIFE) at the bottom of `<body>`, after the `<audio>` element
    - Query `.btn` and `#btn-sound` with guard clause `if (!btn || !clip) return`.
    - Inside the click listener: set `clip.currentTime = 0`, call `clip.play()`.
    - Chain `.catch()` on the play promise: suppress `NotAllowedError` silently; log all other errors with `console.error`.
    - Call `window.open(btn.href, '_blank', 'noopener,noreferrer')` synchronously after `clip.play()` (do not await).
    - Call `e.preventDefault()` to prevent the default anchor from opening a second tab.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 2.3, 4.1_

  - [ ]* 3.2 Write unit tests for the click handler
    - Use a test runner (e.g., Vitest or Jest) with jsdom environment; mock `HTMLAudioElement.play` and `window.open`.
    - U-01: Click when audio not playing → `currentTime === 0`, `play()` called.
    - U-02: Click when `currentTime > 0` → `currentTime` reset to `0` before `play()`.
    - U-03: Single click → `window.open` called with `https://ahmadammar.my.id` and `'_blank'`.
    - U-04: Single click → `window.open` called before the play promise resolves.
    - U-05: `play()` rejects → `window.open` still called; no unhandled rejection.
    - U-06: `play()` rejects with `NotAllowedError` → `console.error` NOT called; `window.open` called.
    - U-07: `play()` rejects with arbitrary error → `console.error` called; `window.open` called.
    - U-08: `btn` or `clip` missing from DOM → no runtime error thrown.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 2.3_

- [ ] 4. Checkpoint — verify core behaviour
  - Ensure all unit tests pass. Open `index.html` in a browser, click the button, and confirm: sound plays, new tab opens, no duplicate tabs appear. Ask the user if any questions arise.

- [ ] 5. Verify preserved button appearance and structural constraints

  - [ ] 5.1 Confirm button attributes and styles are unchanged
    - Inspect the modified `index.html` DOM to confirm `.btn` retains `href`, `target="_blank"`, and `rel="noopener noreferrer"`.
    - Confirm no CSS rules for `.btn` were added or removed.
    - _Requirements: 3.1, 3.2_

  - [ ]* 5.2 Write smoke / structural checks
    - S-01: Assert no `<script src="...">` tags point to external URLs.
    - S-02: Assert the `<audio>` source `src` attribute starts with `data:audio/`.
    - S-03: Assert the repository contains exactly one `.html` file.
    - _Requirements: 5.1, 5.2, 5.3_

- [ ] 6. Final checkpoint — all tests pass
  - Run the full test suite (unit tests + smoke checks). Verify `index.html` opens via `file://` and the button works end-to-end. Ask the user if any questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP.
- No property-based testing is used: the feature is entirely side-effect-driven (`window.open`, `HTMLAudioElement.play`, DOM events), so example-based unit tests and smoke checks provide the right coverage.
- The design document notes a conflict between Requirements 2.2 (suppress visible autoplay errors) and 4.3 (show a visible error when autoplay is blocked). The implementation follows Requirement 2.2 (silent suppression). Requirement 4.3 is not implemented; a non-intrusive toast could be added in a future iteration.
- The Base64 encoding step (task 1.1) is a manual offline step; no code is written here.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["3.1"] },
    { "id": 3, "tasks": ["3.2", "5.1"] },
    { "id": 4, "tasks": ["5.2"] }
  ]
}
```
