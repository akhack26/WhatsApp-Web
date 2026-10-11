from pathlib import Path

content = r"""# Local Personal WhatsApp Voice Assistant

A planned, local-first Python desktop assistant for controlling WhatsApp Web with voice commands on **Zorin OS Linux**. The goal is to support Hindi, Hinglish, and English commands, search existing WhatsApp chats by their displayed names, read incoming messages aloud when enabled, and use an optional local GGUF model to draft replies.

> **Project status:** This README describes the intended design and development plan. Features must not be considered implemented until the corresponding code exists and has been tested. Browser automation and incoming-message detection may need maintenance as WhatsApp Web changes.

## Contents

- [Project goals](#project-goals)
- [Planned features](#planned-features)
- [Safety-first message workflow](#safety-first-message-workflow)
- [Planned architecture](#planned-architecture)
- [Technology choices](#technology-choices)
- [Planned repository structure](#planned-repository-structure)
- [System requirements](#system-requirements)
- [Installation on Zorin OS](#installation-on-zorin-os)
- [Optional local GGUF model](#optional-local-gguf-model)
- [Voice input and speech output](#voice-input-and-speech-output)
- [Security and privacy](#security-and-privacy)
- [Testing strategy](#testing-strategy)
- [Known risks and limitations](#known-risks-and-limitations)
- [Development roadmap](#development-roadmap)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Project goals

The application is intended to provide a personal desktop interface for common WhatsApp Web tasks without requiring a separate database of contact phone numbers.

Example commands the planned assistant should understand:

- “WhatsApp kholo.”
- “Rahul ki chat search karo.”
- “Rahul ko message bhejo: Main 10 minute mein aa raha hoon.”
- “Is message ka reply draft karo.”
- “Message padhkar sunao.”
- “Cancel karo.”

The assistant should show the interpreted command and let the user correct it. Typed input must remain available when a microphone, speech model, or local AI model is unavailable.

## Planned features

### Desktop GUI

- Modern PySide6 desktop interface.
- WhatsApp Web connection and login status.
- Chat/contact-name search.
- Command input and message-draft editor.
- Recipient and message review before sending.
- Optional incoming-message reading and text-to-speech controls.
- Local model configuration and status.
- Clear errors, diagnostics, and privacy controls.

### WhatsApp Web

- Open WhatsApp Web in Chrome or a supported Chromium browser.
- Use a dedicated persistent browser profile for the app.
- Let the user complete QR-code login manually.
- Search chats by the displayed contact or group name.
- Verify the selected chat before preparing a send action.
- Handle missing selectors, timeouts, disconnections, and interface changes explicitly.

The project should not extract or export authentication cookies, passwords, OTPs, or QR authentication data. Browser selectors should be centralized because WhatsApp Web's interface can change without notice.

### Local AI

- Optional GGUF model through `llama.cpp` or `llama-cpp-python`.
- Use local inference for command interpretation, summarization, and reply drafting.
- Validate structured model output before using it.
- Provide a basic parser fallback when no model is configured.
- Keep model weights separate from the source repository.

### Voice

- Pluggable offline speech-recognition interface.
- Hindi, Hinglish, and English are target languages; recognition quality will depend on the selected model and microphone.
- Show the recognized transcript before acting.
- Provide start, stop, mute, and cancel controls.
- Offer text-to-speech for incoming messages when explicitly enabled.
- Keep typed commands available as a fallback.

### Incoming messages

Incoming-message detection should initially be considered **experimental**. If reliable detection cannot be maintained against the current WhatsApp Web interface, the app should display a clear warning instead of silently pretending it is working.

The assistant may prepare a reply using the local model, but the reply must remain a draft until the user explicitly approves it.

## Safety-first message workflow

Every outgoing message must require a fresh, explicit user confirmation. Automatic sending without confirmation is out of scope for the initial design.

Planned state machine:

```text
IDLE
  ↓ create draft
DRAFTED
  ↓ review recipient and text
AWAITING_CONFIRM
  ↓ user confirms
SENDING
  ↓ verify outcome
SENT_CONFIRMED / SENT_UNVERIFIED / FAILED
```

Safety requirements:

1. Show the exact recipient and exact message before sending.
2. Require a new confirmation for each message.
3. Invalidate the approval if the recipient or message changes.
4. Refuse to send to an ambiguous or unverified chat.
5. Keep send capability out of the language model's control.
6. Treat incoming message content as untrusted data.
7. Never claim success unless the result can be verified.

Planned result statuses:

- `SENT_CONFIRMED` — the application found evidence that the outgoing message appeared in the chat and the expected send indicator was present.
- `SENT_UNVERIFIED` — the application cannot reliably confirm the outcome.
- `FAILED` — the send operation failed.

These statuses are design targets and must be implemented and tested before being relied upon.

## Planned architecture

The GUI should not control Playwright directly. Browser automation should run in a dedicated worker thread or a separate asyncio loop, with commands and results exchanged through a queue and Qt signals.

Suggested components:

- **GUI/controller layer:** UI state, command routing, user approvals, and status display.
- **Browser manager:** Browser startup, dedicated profile, login state, and shutdown.
- **WhatsApp driver:** Chat search and carefully verified browser actions.
- **Selector registry:** Centralized selectors with ordered fallbacks and diagnostics.
- **Message manager:** Draft lifecycle, approval state machine, and send-result status.
- **Local AI adapter:** Optional GGUF model loading and validated outputs.
- **Speech recognizer:** Pluggable offline transcription.
- **Speech output:** Pluggable TTS with a fallback engine where available.
- **Configuration/settings:** Versioned settings and migrations.
- **Logger:** Centralized redaction; message content should not be logged by default.
- **Fake driver:** Test implementation that never connects to a real WhatsApp account.

## Technology choices

| Area | Planned technology | Notes |
|---|---|---|
| Language | Python | Main application language |
| GUI | PySide6 | Desktop interface |
| Browser automation | Playwright | Visible browser; selectors may need updates |
| Local LLM | llama.cpp / llama-cpp-python | Optional; requires a compatible GGUF file |
| Speech recognition | Vosk or another offline engine | Hindi/Hinglish accuracy varies |
| Alternative speech recognition | faster-whisper, if suitable locally | Optional and hardware-dependent |
| Text-to-speech | Piper or `espeak-ng` fallback | Voice availability depends on installed packages |
| Settings | SQLite or validated local config | Avoid storing unnecessary personal data |
| Tests | pytest and a fake WhatsApp driver | Automated tests must not send real messages |

## Planned repository structure

The implementation may evolve, but a modular layout is recommended:

```text
local-whatsapp-assistant/
├── main.py
├── README.md
├── requirements.txt
├── requirements-ai.txt
├── requirements.lock
├── .gitignore
├── .env.example
├── install.sh
├── run.sh
├── assistant/
│   ├── __init__.py
│   ├── gui.py
│   ├── controller.py
│   ├── config.py
│   ├── browser_manager.py
│   ├── whatsapp.py
│   ├── selectors.py
│   ├── message_manager.py
│   ├── local_ai.py
│   ├── command_parser.py
│   ├── voice_input.py
│   ├── speech_output.py
│   ├── security.py
│   └── logger.py
├── tests/
│   ├── test_command_parser.py
│   ├── test_message_approval.py
│   ├── test_config.py
│   ├── test_log_redaction.py
│   └── test_fake_driver.py
├── docs/
│   ├── ARCHITECTURE.md
│   ├── INSTALLATION.md
│   └── TROUBLESHOOTING.md
└── data/                  # Local private data; never commit
    └── browser-profile/   # Persistent WhatsApp Web session; never share
```

Only add files that are actually implemented. Do not commit `data/`, browser profiles, credentials, virtual environments, logs containing private data, or model weights.

## System requirements

- Zorin OS Linux.
- A compatible Python version. The design forecast targets Python 3.10–3.12; the actual supported version must be confirmed by testing.
- Chrome or a Playwright-supported Chromium browser.
- Internet access for WhatsApp Web and initial dependency/model downloads.
- Microphone and audio output for voice features.
- Enough RAM and storage for the chosen GGUF model if local AI is enabled.

A GGUF model can require several gigabytes of storage and memory. Choose model size and quantization based on the computer's actual RAM and CPU. GPU acceleration is optional and depends on the installed build.

## Installation on Zorin OS

The following is a **proposed setup flow** for the implementation. Run it only after the repository contains the corresponding scripts and dependency files.

### 1. Install system packages

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip
```

For optional audio features, additional system libraries may be required. For example, `sounddevice` commonly needs PortAudio. The installer should check dependencies and print the exact package command if something is missing; it should not silently run privileged installation commands.

### 2. Create a virtual environment

From the project directory:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 3. Install application dependencies

```bash
pip install -r requirements.txt
python -m playwright install chromium
```

Install optional AI dependencies separately only if needed:

```bash
pip install -r requirements-ai.txt
```

Native packages such as `llama-cpp-python` may need a compiler, CMake, and a build compatible with the computer. If installation fails, keep the core GUI usable and consult the error output rather than assuming the model backend installed successfully.

### 4. Launch

If the scripts have been implemented:

```bash
./run.sh
```

Otherwise, use the entry point documented by the actual source code. The README must be updated when implementation details change.

### 5. Connect WhatsApp Web

1. Open WhatsApp Web from the app.
2. Scan the QR code manually with your own WhatsApp account.
3. Verify that the expected account and chat list are visible.
4. Search a chat by its displayed name.
5. Confirm the selected recipient and exact draft before any send attempt.

Do not share the browser profile or copy it into a public repository.

## Optional local GGUF model

The app is designed to work without a GGUF model. To enable local inference:

1. Select a GGUF model that supports your intended language and task.
2. Check the model's license and download it from a source you trust.
3. Store it outside the Git repository, for example under `~/Models/`.
4. Configure the model path through the app's settings or documented environment variable.
5. Test model loading and response quality before using it for message drafts.

Example environment variable convention:

```bash
export GGUF_MODEL_PATH="$HOME/Models/your-model.gguf"
```

This variable is only a proposed convention until it is implemented in the application. Do not commit model files to GitHub. Never give a language model direct authority to send messages, execute shell commands, or access arbitrary files.

## Voice input and speech output

Speech engines and language models have different language coverage and resource requirements. Hindi and Hinglish recognition may be imperfect, especially for names, mixed scripts, and noisy audio.

Recommended behavior:

- Always display the recognized transcript.
- Let the user correct it before creating a draft.
- Keep typed input available.
- Provide a clear error if a speech model is missing.
- Make listening and TTS easy to stop.
- Do not treat recognized words as permission to send a message.

For TTS, the implementation may use Piper where a compatible voice is available, or `espeak-ng` as a fallback. Actual Hindi voice availability must be verified for the chosen installation.

## Security and privacy

This application handles private messages and an authenticated browser session. Treat the browser profile as sensitive account data.

- Keep the browser profile local and restrict its permissions, ideally to the current Linux user (`chmod 700` on the profile directory).
- Do not commit session data, cookies, QR codes, credentials, OTPs, or private logs.
- Do not send message contents to a cloud AI service by default.
- Do not log full message bodies by default.
- Centralize log redaction and test that private message text is not written to logs.
- Treat incoming messages as untrusted text; they may contain instructions intended to manipulate an AI model.
- Validate model output against a strict schema and an allow-list of supported intents.
- Never execute code or shell commands generated by the model.
- Keep confirmation and sending authority in the application layer, outside the model.
- Provide a clear way to stop the assistant and close the browser.

“Local-first” does not mean every operation is offline: WhatsApp Web requires a network connection, and downloading dependencies or models also uses the network.

## Testing strategy

Automated tests should use a fake WhatsApp driver and must not interact with real chats.

Planned test coverage:

- Hindi, Romanized Hindi, Hinglish, and English command parsing.
- Recipient/message extraction and ambiguous commands.
- Missing or invalid model paths.
- Approval-state transitions.
- Approval invalidation after recipient or message changes.
- Cancel behavior.
- Browser timeouts and disconnected sessions.
- Selector self-check and diagnostics.
- Log redaction.
- Settings schema migrations.
- Application startup without an AI model or microphone.

A manual integration checklist should separately cover browser login, chat search, recipient verification, message drafting, send-result verification, incoming-message detection, speech input, and TTS.

## Known risks and limitations

- **WhatsApp Web changes:** DOM selectors can break and may require updates.
- **Automation restrictions:** Automating WhatsApp Web may conflict with WhatsApp's terms or policies and may lead to account restrictions. Human confirmation reduces some risks but does not remove them.
- **Message verification:** A UI action alone does not prove a message was delivered. The app must distinguish confirmed, unverified, and failed outcomes.
- **Speech accuracy:** Hindi/Hinglish speech recognition and contact-name recognition may be imperfect.
- **Incoming-message detection:** DOM polling or observation may be brittle and should be marked experimental until tested.
- **GGUF installation:** Native builds may fail or need CPU/GPU-specific configuration.
- **Model quality:** A local model may misunderstand commands or draft inaccurate replies; review all text before sending.
- **Browser session security:** Anyone with access to an unlocked browser profile may be able to access the logged-in session.

This project is not affiliated with, endorsed by, or officially supported by WhatsApp or Meta.

## Development roadmap

### Phase 1 — Foundation
- [ ] Read and implement the full project specification.
- [ ] Create modular project structure and PySide6 GUI.
- [ ] Add configuration, error handling, and logging.
- [ ] Add installer and launcher scripts.

### Phase 2 — Browser integration
- [ ] Dedicated browser worker and persistent profile.
- [ ] Manual QR login flow.
- [ ] Chat search by displayed name.
- [ ] Centralized selectors and diagnostics.
- [ ] Browser failure and timeout handling.

### Phase 3 — Safe messaging
- [ ] Message draft editor.
- [ ] Explicit recipient and message review.
- [ ] Approval state machine and approval hash.
- [ ] Cancel and invalidation behavior.
- [ ] Verified send-result reporting.
- [ ] Fake driver for tests.

### Phase 4 — Local AI
- [ ] Optional GGUF backend.
- [ ] Model path configuration and validation.
- [ ] Strict output validation and fallback parser.
- [ ] Prompt-injection protections.
- [ ] Tests without model weights.

### Phase 5 — Voice and incoming messages
- [ ] Offline speech-recognition interface.
- [ ] Transcript review and typed fallback.
- [ ] TTS interface and fallback.
- [ ] Experimental incoming-message detection.
- [ ] Deduplication and stop controls.

### Phase 6 — Quality and release
- [ ] Automated tests and syntax checks.
- [ ] Zorin OS installation verification.
- [ ] Dependency pinning and tested-version documentation.
- [ ] Privacy/security review.
- [ ] Troubleshooting guide.
- [ ] Clean ZIP release without private data or model weights.

## Troubleshooting

### WhatsApp Web does not open
Check whether Chrome or Chromium is installed, whether Playwright's browser is installed, and whether the app reports a browser launch error.

### QR login appears every time
Confirm the app is using the same dedicated persistent browser profile and that the current Linux user can read and write it. Do not copy or share that profile.

### Chat search fails
WhatsApp Web's visible labels and selectors can change. Use the app's diagnostic output, redact private information before sharing logs, and update the centralized selector registry.

### GGUF model fails to load
Check the model path, available memory, Python version, and compatibility of the installed `llama-cpp-python` build. The core GUI should still start without the model.

### Microphone or TTS does not work
Check Linux microphone permissions and audio devices. Confirm the required system audio libraries and the selected offline speech model are installed.

### A message outcome is unclear
Do not automatically retry: a previous attempt may have succeeded even if confirmation failed. Inspect the conversation manually before attempting to send again.

## Contributing

Contributions are welcome. Before submitting changes:

1. Keep browser, GUI, AI, voice, and message-approval logic separated.
2. Add tests for new parser phrases and safety-state changes.
3. Use a fake driver in automated tests.
4. Do not include private chats, browser profiles, credentials, or model weights in commits.
5. Update documentation to distinguish implemented, experimental, and planned features.

## License

Choose and add a license before publishing the repository. Do not assume the project is open-source licensed until a `LICENSE` file has been added. Verify the licenses of all dependencies, speech models, and GGUF models separately.
"""

path = Path("/mnt/data/README.md")
path.write_text(content, encoding="utf-8")
print(f"Created {path} ({len(content.splitlines())} lines, {path.stat().st_size} bytes)")
