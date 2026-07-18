# Build Log

## 1. Storage: RAID configuration

Disks arrived unconfigured on the HPE P408i-p controller. Since it's a hardware
RAID controller (no true HBA passthrough mode), the OS installer can't see raw
disks until a logical drive exists.

- Configured via **SSA (F10 at boot)** before touching the OS installer
- **RAID 1 across 2 of 5 available SAS drives** — redundancy for a box intended
  to run daily, while leaving 3 drives powered off (less power draw, less wear,
  quieter)
- Storage budget: ~150-200GB is comfortable (OS ~20-30GB, models ~40-60GB,
  Hermes Agent + logs/memory ~15GB, swap ~8GB)

## 2. OS install: Ubuntu 24.04 LTS

- ISO hosted on Services LXC via existing nginx setup (`/var/www/iso/`), mounted
  to the server via iLO 5 virtual media (Mount via URL, boot on next reset, F11
  for one-time boot menu)
- Network: single active NIC (`eno1`, DHCP) — second port unused, "autoconfiguration
  failed" on it is expected/harmless
- No proxy configured (not part of this homelab's setup)
- **Deselected all snap packages** in the installer (microk8s, nextcloud, wekan,
  canonical-livepatch, rocketchat-server all pre-checked by default) — none needed,
  each adds background overhead competing with inference for RAM/CPU

## 3. Power configuration — found and fixed before benchmarking

iLO **Power Regulator** was set to **Dynamic Power Savings Mode** by default, which
throttles P-states at the firmware level (below what the OS governor can override).
Switched to **Static High Performance Mode**. Power Capping confirmed disabled via
ROM option. (Full investigation into whether this actually mattered for inference
speed — it didn't — is in `03-benchmark.md`.)

## 4. Ollama install

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama --version   # 0.31.2
sudo systemctl status ollama
```

No GPU detected (expected) — runs CPU-only.

```bash
ollama pull qwen2.5:7b
```

## 5. Hermes Agent install

Installed under a **dedicated system user**, not the personal login account:

```bash
sudo adduser --system --group --home /home/hermes-rafa --shell /bin/bash hermes-rafa
sudo -u hermes-rafa -i
pip install hermes-agent --break-system-packages
```

Note: `sudo -u hermes-rafa -i` (using the *operator's* sudo credentials) is the
right way to drop into a service account's shell — plain `su - hermes-rafa` fails
because the service account has no password set, and shouldn't need one.

**PATH issue** — pip installs user scripts to `~/.local/bin`, which isn't on PATH
by default. Same fix needed once per Linux user:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
hermes --version   # v0.18.2
```

System-level installs (Node.js, Chromium runtime libs, ripgrep — needed for Browser
Automation and Skills tools) were run from the operator account (`capepp`) with
`sudo`, not from inside `hermes-rafa` — package installs are system-wide regardless
of which user triggers them, and the service account intentionally has no sudo
rights.

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
sudo apt install -y libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 \
  libxcomposite1 libxdamage1 libxfixes3 libxrandr2 libgbm1 libasound2t64
sudo apt install -y ripgrep
```

## 6. Hermes Agent setup wizard (`hermes setup`)

Choices made during **Full Setup** (not Quick Setup/Nous Portal, not Blank Slate):

| Prompt              | Choice                                                   | Why |
|---------------------|----------------------------------------------------------|-------------------------------------------------|
| Model provider      | Custom → `http://127.0.0.1:11434/v1`, model `qwen2.5:7b` | Local Ollama, not a cloud provider              |
| Context length      | Auto-detect                                              | See open issue below                            |
| Display name        | "Ready Room"                                             | Ties to homelab project branding                |
| Terminal backend    | Local                                                    | Agent, model, and data all stay on this machine |
| Messaging platforms | Skip all                                                 | No third-party platform in the access path      |
| Tools               | Trimmed from 18 defaults down to 6                       | Every enabled tool adds to the system prompt injected on every session (see `03-benchmark.md` for the real cost this has) — Computer Use turned off outright (headless server, no display)                                       |
| Browser automation  | Local Browser (headless Chromium)                        | Free, local, no cloud dependency                |
| Image generation    | Skip                                                     | No local option exists; not a stated goal       |

## 7. Open issue: model context window

Hermes Agent requires a minimum 64K token context window; Ollama auto-detected
Qwen 2.5 7B's window as 32,768 and refused to start.

Attempted fix: baked a higher `num_ctx` into a custom Ollama tag —

```bash
cat << 'EOF' > Modelfile
FROM qwen2.5:7b
PARAMETER num_ctx 65536
EOF
ollama create qwen2.5-64k -f Modelfile
```

**Result: did not work.** Ollama's own logs show it rejecting the override:

```
level=WARN msg="requested context size too large for model" num_ctx=65536 n_ctx_train=32768
```

and launching with `-c 32768` regardless. This particular GGUF has no YaRN
scaling baked in (`rope scaling = linear`, `n_ctx_orig_yarn = 32768` — same as
trained length), so the model genuinely wasn't built to extend past 32K.
Setting `context_length: 65536` in `~/.hermes/config.yaml` satisfies Hermes
Agent's *startup check* but does not change what Ollama actually serves — a
cosmetic fix, not a real one.

**Two real paths identified, not yet executed:**
1. Pull `org/qwen2.5-1m:7b` — Qwen's separately-trained, genuinely long-context
   variant (dual chunk + sparse attention, not just a config flag)
2. Force rope-scaling extrapolation on the current model via `PARAMETER
   rope_scaling yarn` — technically possible, but pushes the model past its
   trained range with an open question on coherence past 32K

Decision pending as of this write-up.

## 8. Tool selection (final, 6 enabled)

Trimmed from Hermes Agent's 18 default tools after discovering the injected
system prompt (persona + full tool definitions) directly drives both first-message
latency and per-token generation cost for the entire session (see `03-benchmark.md`).
Kept: file operations, terminal/code execution, web search, memory, task planning,
skills. Cut: everything platform-specific (Discord, Spotify, Home Assistant, X
search, video/image generation, TTS, Computer Use).
