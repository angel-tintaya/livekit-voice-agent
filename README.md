# LiveKit Voice Agent

A minimal demo of a real-time voice agent built with [LiveKit Agents](https://docs.livekit.io/agents/). The agent joins a LiveKit room, listens with speech-to-text, reasons with an LLM, and replies with text-to-speech.

This document records the steps used to create the project and is the starting point for running it locally.

## Prerequisites

- Python 3.11 or later
- [uv](https://docs.astral.sh/uv/) for project and dependency management
- A [LiveKit Cloud](https://cloud.livekit.io/) project (or a self-hosted LiveKit server)

## 1. Create the project

Initialize a bare Python project (no sample application files) and move into the project directory:

```bash
uv init livekit-voice-agent --bare
cd livekit-voice-agent
```

`--bare` creates a `pyproject.toml` without scaffolding extra files, which keeps this demo small and explicit.

## 2. Install dependencies

Add the LiveKit Agents SDK with the Silero VAD and turn-detector extras, then the noise-cancellation plugin and dotenv support:

```bash
uv add "livekit-agents[silero,turn-detector]~=1.3"
uv add "livekit-plugins-noise-cancellation~=0.2"
uv add python-dotenv
```

These packages provide:

| Package | Role |
| --- | --- |
| `livekit-agents` | Agent runtime, session lifecycle, and CLI |
| `silero` extra | Voice activity detection (VAD) |
| `turn-detector` extra | End-of-turn detection for natural conversation |
| `livekit-plugins-noise-cancellation` | Background-voice cancellation on the audio input |
| `python-dotenv` | Loads credentials from a local `.env` file |

## 3. Configure LiveKit credentials

Copy the example environment file and fill in the values from your [LiveKit Cloud](https://cloud.livekit.io/) project:

```bash
cp .env.example .env
```

On Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Then replace the placeholders in `.env` with your project's URL, API key, and API secret. The expected format is in [`.env.example`](.env.example):

```env
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=your_api_key
LIVEKIT_API_SECRET=your_api_secret
```

`.env` is listed in `.gitignore` and must not be committed. It contains secrets that authenticate the agent to your LiveKit project. Commit `.env.example` only; it is a template with no real credentials.

## 4. Create the agent

Add `agent.py` with a basic configuration:

- An `Assistant` that extends `Agent` and sets system instructions
- An `AgentSession` wired to STT, LLM, TTS, Silero VAD, and the multilingual turn detector
- Noise cancellation on the room audio input
- A CLI entrypoint so the agent can run with `uv run agent.py`

The demo uses:

- **STT:** AssemblyAI Universal Streaming (`en`)
- **LLM:** OpenAI GPT-4.1 mini
- **TTS:** Cartesia Sonic 3
- **VAD:** Silero
- **Turn detection:** LiveKit multilingual end-of-turn model (`MultilingualModel`)

## 5. Download model files (first run only)

The Silero VAD (and related local models) must be downloaded once before the agent can start:

```bash
uv run agent.py download-files
```

`download-files` only fetches models for plugins that `agent.py` currently imports. Re-run it after you add a plugin that ships local model files (for example the multilingual turn detector in [step 8](#8-enable-multilingual-turn-detection)). Also re-run it if you reinstall dependencies or clear the model cache.

## 6. Run the agent

Start the agent in console mode for local testing (microphone in, speaker out):

```bash
uv run agent.py console
```

Speak into the microphone. The agent transcribes your speech, generates a reply, and plays it back.

## 7. Stop the agent

Press `Ctrl+C` in the terminal to shut the agent down cleanly.

## 8. Enable multilingual turn detection

The first version of the agent used Silero VAD only. To detect when the user has finished speaking with a semantic end-of-turn model, add the turn-detector extra, wire it into `agent.py`, then download the model files.

### Install the extra

This extra provides the multilingual model referenced in the agent code:

```bash
uv add "livekit-agents[turn-detector]"
```

If you already installed `livekit-agents[silero,turn-detector]` in [step 2](#2-install-dependencies), this command is idempotent. Run it whenever the extra was not part of the original install.

### Update the agent

In `agent.py`, import the multilingual model and pass it to the session:

```python
from livekit.plugins.turn_detector.multilingual import MultilingualModel

session = AgentSession(
    stt="assemblyai/universal-streaming:en",
    llm="openai/gpt-4.1-mini",
    tts="cartesia/sonic-3",
    vad=silero.VAD.load(),
    turn_detection=MultilingualModel(),
)
```

### Download the turn-detector files

Installing the extra is not enough. The plugin loads a local Hugging Face model (`livekit/turn-detector`, revision `v0.4.1-intl`). Those files are fetched only by `download-files`, and only after `agent.py` imports the plugin:

```bash
uv run agent.py download-files
```

If you skip this and run console immediately, the agent fails with:

```text
Could not find model livekit/turn-detector with revision v0.4.1-intl.
Make sure you have downloaded the model before running the agent.
Use `python -m livekit.agents download-files` to download the models.

RuntimeError: livekit-plugins-turn-detector initialization failed.
```

### Run the agent again

```bash
uv run agent.py console
```

On startup you may see:

```text
[transformers] PyTorch was not found. Models won't be available and only tokenizers, configuration and file/data utilities can be used.
INFO:livekit.agents:process initialized
```

This is expected and is not an error. Hugging Face `transformers` prints that line because PyTorch is not installed. The turn detector uses `transformers` only for tokenization and runs inference with ONNX Runtime, so PyTorch is not required. `process initialized` means the agent started successfully. Do not install PyTorch just to hide the message.
