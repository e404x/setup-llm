# Run OpenCode with a local LLM on Ubuntu

This guide matches the OpenCode v2 setup tested on this laptop. It runs Qwen3 8B through Ollama at `127.0.0.1:11434`. Local inference needs no API key and has no provider usage quota. Downloading the software and model requires internet access. OpenCode tools and plugins can make their own network requests.

## 1. Check what is installed

```sh
command -v ollama opencode
ollama --version
opencode --version
nvidia-smi
```

On this laptop, both programs are already installed. Ollama runs as a user service. Start with step 3 to check it, or skip to step 6 to use it.

## 2. Install Ollama on a fresh Ubuntu machine

Run the official installer and enable its system service.

```sh
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama
```

For a temporary terminal-based setup, run `OLLAMA_HOST=127.0.0.1:11434 OLLAMA_NO_CLOUD=1 OLLAMA_CONTEXT_LENGTH=16384 ollama serve` in its own terminal. Use one Ollama server at a time because the service and manual launch use the same port. [Ollama Linux installation](https://docs.ollama.com/linux)

## 3. Configure Ollama and download the model

For a **fresh system service**, run `sudo systemctl edit ollama` and add these lines to the editable portion.

```ini
[Service]
Environment=OLLAMA_HOST=127.0.0.1:11434
Environment=OLLAMA_NO_CLOUD=1
Environment=OLLAMA_CONTEXT_LENGTH=16384
```

Then run `sudo systemctl daemon-reload` and `sudo systemctl restart ollama`.

This laptop uses a **user service** with those values already configured. Use `systemctl --user status ollama` here and `systemctl --user restart ollama` if you change its settings.

Check the API, download the model, and test it directly.

```sh
curl -fsS http://127.0.0.1:11434/api/version
ollama pull qwen3:8b
ollama list
ollama run qwen3:8b "What is 17 plus 25?"
ollama ps
```

While the model is loaded, `ollama ps` shows processor placement and context. This laptop reached `100% GPU` and `16384` context on its RTX 5050. That window includes instructions, tool definitions, conversation history, file excerpts, and generated tokens. Larger repositories benefit from focused tasks and fresh sessions. [Ollama context length](https://docs.ollama.com/context-length)

## 4. Install OpenCode v2

On a fresh machine, use the official v2 installer, then check the version. Open a new terminal if the command is missing after installation.

```sh
curl -fsSL https://opencode.ai/v2/install | bash
opencode --version
```

[OpenCode v2 installation](https://opencode.ai/docs/)

## 5. Make the local model the OpenCode default

Create the config directory and back up any existing configuration.

```sh
mkdir -p ~/.config/opencode
if [ -f ~/.config/opencode/opencode.json ]; then
  cp -p ~/.config/opencode/opencode.json \
    ~/.config/opencode/opencode.json.backup-"$(date +%Y%m%d-%H%M%S)"
fi
```

Open `~/.config/opencode/opencode.json` in your editor. For an existing file, set the top-level `model` and merge the `ollama` entry under `providers`, keeping your other settings. A minimal complete v2 configuration is below.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "ollama/qwen3:8b",
  "providers": {
    "ollama": {
      "settings": {
        "baseURL": "http://127.0.0.1:11434/v1"
      },
      "models": {
        "qwen3:8b": {
          "name": "Qwen3 8B (local)",
          "capabilities": {
            "tools": true,
            "input": ["text"],
            "output": ["text"]
          },
          "limit": {
            "context": 16384,
            "output": 4096
          }
        }
      }
    }
  }
}
```

Run `opencode reload` if OpenCode's background server is already running. The configuration uses the v2 `providers` and `settings` syntax. [OpenCode provider configuration](https://opencode.ai/docs/providers/)

## 6. Verify and use OpenCode

From the directory of a project you want OpenCode to work with, run these commands.

```sh
opencode models | grep -F 'ollama/qwen3:8b'
opencode run "What is 17 plus 25? Answer in one line."
opencode
```

The one-off prompt should answer 42. In the interactive session, ask OpenCode to read a project file and summarize it to test tool use. To select the local model explicitly for one run, use `opencode run -m ollama/qwen3:8b "Explain this project"`.

## 7. Review an app defensively

Open a local checkout of an app you own or are authorized to review. Start with a source-only prompt and name the relevant directory.

> Review the authentication and authorization code in this repository. Identify missing server-side access checks, unsafe defaults, and sensitive data in logs. For each finding, cite the file and line, explain the impact, and propose a fix and a regression test. Read the source only and ask before editing files or making network requests.

Review the cited code yourself, then ask OpenCode to apply the fix and run the project's existing tests. A useful authorization regression test checks that a permitted user can perform an action and that a user without the required permission receives the expected denial.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Connection refused | Start the correct Ollama service and repeat the API version check. On this laptop use `systemctl --user start ollama`. |
| Address already in use | Run `ss -lntp | grep 11434` and check which instance owns the port. |
| OpenCode lists the model but a prompt fails | Test `ollama run qwen3:8b "Say hello"` first, then check that OpenCode's endpoint ends in `/v1` and reload the config. |
| Responses are slow | Check `ollama ps` while generating and run `nvidia-smi` to check GPU memory. |
| Long tasks lose context | Start a fresh session and work on one part of the repository at a time. If changing the context size, update both the Ollama service and OpenCode's model limit. |
| Need Ollama logs | On this laptop run `journalctl --user -u ollama -n 50 --no-pager`. For a system service run `sudo journalctl -u ollama -n 50 --no-pager`. |
