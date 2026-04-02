# VoiceInput

macOS menu-bar voice input app. Hold Fn to record, release to transcribe and inject text into the focused input field.

## Tech Stack

- **Language:** Swift 5.9, macOS 14+ (Sonoma)
- **Build:** Swift Package Manager + Makefile
- **Frameworks:** AppKit, Speech (SFSpeechRecognizer), AVFoundation, Carbon (TIS input source APIs)
- **Architecture:** LSUIElement app (menu bar only, no Dock icon), single-target executable

## Build & Run

```bash
make build    # Build release .app bundle with ad-hoc signing
make run      # Build and launch
make install  # Copy to /Applications
make clean    # Remove build artifacts
```

## Project Structure

```
Sources/VoiceInput/
  main.swift          # Entry point, sets up NSApplication in .accessory mode
  AppDelegate.swift   # Menu bar setup, Fn key → record/stop flow, speech callbacks, LLM integration
  KeyMonitor.swift    # CGEvent tap on flagsChanged to capture Fn key (suppresses emoji picker) + right Ctrl key (custom branch)
  SpeechEngine.swift  # SFSpeechRecognizer streaming recognition + audio RMS level metering
  OverlayPanel.swift  # Frameless capsule floating window (NSPanel + NSVisualEffectView) with waveform bars
  TextInjector.swift  # Clipboard + simulated Cmd+V paste, with CJK input method workaround
  LLMRefiner.swift    # OpenAI-compatible API client for post-transcription error correction (singleton)
  SettingsWindow.swift # LLM settings panel (API Base URL, API Key, Model)
```

## Key Design Decisions

- **Fn key monitoring:** Uses `CGEvent.tapCreate` with `.defaultTap` to intercept and suppress Fn flagsChanged events, preventing the emoji picker from appearing.
- **CJK paste workaround:** `TextInjector` detects non-ASCII input sources via TIS APIs and temporarily switches to ABC/US keyboard before simulating Cmd+V, then restores the original input source.
- **Clipboard preservation:** Original clipboard content is saved before injection and restored after a 500ms delay.
- **LLM refinement:** Conservative system prompt — only fixes obvious speech recognition errors (homophones, English terms misrecognized as Chinese). Never rewrites or polishes text.
- **Audio waveform:** 5 bars with weights `[0.5, 0.8, 1.0, 0.75, 0.55]`, driven by real-time audio RMS. Smoothed envelope with 40% attack / 15% release + random jitter.
- **Default language:** zh-CN (Simplified Chinese). Language selection persisted in UserDefaults.

## Conventions

- No external dependencies — pure Apple frameworks only
- All UI runs on main thread; speech callbacks dispatch to main
- UserDefaults keys: `selectedLocaleCode`, `llmEnabled`, `llmAPIBaseURL`, `llmAPIKey`, `llmModel`
- Bundle identifier: `com.yetone.VoiceInput`
- Log file: `~/Library/Logs/VoiceInput.log` (LLM requests/responses)

## Git & PR Workflow

- **Upstream:** `origin` → `yetone/voice-input-dist`
- **Fork:** `myfork` → `leolovenet/voice-input-dist`
- **Branches:**
  - `main` — kept in sync with upstream, never commit directly
  - `custom` — personal tweaks (e.g. right Ctrl trigger), push to fork only, never PR. **Daily use branch.**
  - Feature branches (e.g. `fix/overlay-text`) — created from `main` for each PR
- **PR workflow:**
  ```bash
  git checkout main && git pull origin main
  git checkout -b fix/description
  # make changes, commit
  git push myfork fix/description
  gh pr create --repo yetone/voice-input-dist --head leolovenet:fix/description --base main --title "..." --body "..."
  ```
- **After PR merged:** clean up and rebase `custom`:
  ```bash
  git checkout main && git pull origin main
  git branch -d fix/description
  git push myfork --delete fix/description
  git checkout custom && git rebase main
  ```

## Required Permissions

- **Accessibility:** For CGEvent tap (Fn key monitoring)
- **Microphone:** For audio recording
- **Speech Recognition:** For SFSpeechRecognizer
