# WhatsApp-Web

Technical Debt Forecast: Local WhatsApp Voice Assistant
Prepared by Claude, October 11, 2026. Based on the project specification only (no code audited yet).
Purpose
The project spec describes a local, voice-controlled WhatsApp Web assistant for Zorin OS, built with PySide6, Playwright, llama-cpp-python, Vosk and Piper. There is no code yet, so this document forecasts where technical debt is most likely to build up. Once code exists, it can be audited against this same list.
P0: Design these in from day one
• Qt and Playwright threading clash
    ◦ Why: PySide6 runs its own event loop. Playwright's sync API breaks inside it, and its async API needs a separate asyncio loop.
    ◦ Fix: Run Playwright in one dedicated worker thread (or a QThread with its own asyncio loop). Talk to it only through a command queue and Qt signals. Never touch the browser from the GUI thread.
• WhatsApp DOM selectors
    ◦ Why: They will break without notice, and every break forces a hotfix.
    ◦ Fix: Keep all selectors in one selectors.py with ordered fallbacks, preferring ARIA roles, data-testid and visible text over class names. Add a startup self-check that reports which selectors are missing. Ship a --diagnose command that saves a redacted DOM snapshot for debugging.
• Send verification
    ◦ Why: "Sent" status is hard to confirm, and the spec forbids false success reports.
    ◦ Fix: Model send results as SENT_CONFIRMED, SENT_UNVERIFIED or FAILED. Only report "sent" when the message appears in the chat with an outgoing tick, and say "unverified" otherwise.
• Approval state machine
    ◦ Why: Scattered boolean flags are where "sent to the wrong person" bugs come from.
    ◦ Fix: Implement one small state machine (IDLE → DRAFTED → AWAITING_CONFIRM → SENDING) in message_manager.py. Bind each approval to a hash of the recipient and exact text, and invalidate it on any change. The browser layer should refuse to send without a valid token.
• Prompt injection from incoming messages
    ◦ Why: Incoming WhatsApp text is untrusted input going into an LLM.
    ◦ Fix: The LLM should only return JSON validated against a strict schema with an allow-list of intents. Wrap message text as quoted data. Keep the send capability out of the LLM's reach entirely, so nothing the model outputs can trigger a send.
P1: High risk
• llama-cpp-python install fragility
    ◦ Why: It compiles native code, so it needs cmake and a compiler, and the build is sensitive to your CPU. A failed build blocks the whole install.
    ◦ Fix: Make it an optional extra (requirements-ai.txt). Have install.sh check for build-essential and cmake and give a clear message if they are missing. Pin the version and keep the regex parser as a fallback.
• Hindi and Hinglish speech recognition quality
    ◦ Why: Vosk has a small Hindi model and an Indian English model, but no Hinglish model. Accuracy on names and mixed-language commands will be modest.
    ◦ Fix: Put speech-to-text behind an interface, such as SpeechRecognizer.transcribe(), so engines can be swapped later (for example faster-whisper locally). Always show the transcript for confirmation, and keep the typed command as the primary fallback.
• Command parser sprawl
    ◦ Why: Hinglish phrasing varies a lot. A regex pile grows without bound and becomes unmaintainable.
    ◦ Fix: Use a table-driven parser with intent patterns in data. Add a test fixture file of phrases (including Devanagari and Roman variants) so every new phrasing becomes one test case.
• Incoming-message detection
    ◦ Why: Polling the DOM is the most brittle feature and will often be unreliable.
    ◦ Fix: Use a MutationObserver or a low-frequency poll, deduplicate by (chat, message id or hash), and label the feature "experimental". Report the limitation in the UI when detection fails instead of failing silently.
P2: Medium risk
• Audio stack dependencies: sounddevice needs the system libportaudio2, and Piper needs playback. Have install.sh detect these and print the exact apt command for you to run yourself, since the spec forbids it from using sudo.
• Piper Hindi voice availability: Check which Hindi voices exist for your Piper version. Hide the TTS engine behind an interface and fall back to espeak-ng if needed.
• Playwright and Chrome version drift: Use channel="chrome" with a persistent context, and add a startup check that falls back to Playwright's bundled Chromium with a clear message.
• No abstraction for testability: Define a WhatsAppDriver interface with a FakeDriver for tests. Without it, tests either touch real WhatsApp or mock deep Playwright internals, which breaks easily.
• Log redaction: Centralize redaction in logger.py with a filter, and add a test that fails if message text appears in logs. Scattered .replace() calls leak eventually.
• Settings schema drift: Keep a schema_version in SQLite settings and a migration function from the start.
P3: Lower risk, but plan for them
• Dependency pinning: Provide a requirements.lock with hashes, and document the tested Python version (3.10 to 3.12).
• Profile security: Create the browser profile directory with chmod 700, check permissions at startup, and warn on drift.
• GUI and logic coupling: The GUI should call a controller layer, never modules directly, so the app can run headless or in CLI mode.
• Docs rot: Generate the troubleshooting selector section from the --diagnose output format, so docs and code cannot diverge.
Risk outside the code
Automating WhatsApp Web may conflict with WhatsApp's terms of service, and accounts can be restricted for automated behavior. The human-in-the-loop send rule lowers that risk but does not remove it. The README should say so.
Next steps
1. Upload the project folder or ZIP (no model weights, no browser profile) for a file-level audit against this checklist.
2. Share your Zorin OS version and RAM/CPU, which affects the model and library recommendations.
