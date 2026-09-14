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
- An `AgentSession` wired to STT, LLM, TTS, and Silero VAD
- Noise cancellation on the room audio input
- A CLI entrypoint so the agent can run with `uv run agent.py`

The demo uses:

- **STT:** AssemblyAI Universal Streaming (`en`)
- **LLM:** OpenAI GPT-4.1 mini
- **TTS:** Cartesia Sonic 3
- **VAD:** Silero

## 5. Download model files (first run only)

The Silero VAD (and related local models) must be downloaded once before the agent can start:

```bash
uv run agent.py download-files
```

Skip this step on later runs unless you reinstall dependencies or clear the model cache.

## 6. Run the agent

Start the agent in console mode for local testing (microphone in, speaker out):

```bash
uv run agent.py console
```

Speak into the microphone. The agent transcribes your speech, generates a reply, and plays it back.

## 7. Stop the agent

Press `Ctrl+C` in the terminal to shut the agent down cleanly.
