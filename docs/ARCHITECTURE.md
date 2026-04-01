# MetaClaw Multi-Provider Routing Architecture

Last updated: 2026-04-01

## 1. OVERVIEW

This is a fork of [aiming-lab/MetaClaw](https://github.com/aiming-lab/MetaClaw) with multi-provider LLM routing added. Upstream MetaClaw is an RL-based agent training framework with a built-in proxy server. This fork extends the proxy to act as a unified gateway that routes requests to multiple LLM providers from a single endpoint.

### What this fork adds

- **Multi-provider routing** — One proxy (localhost:30000) routes to MiniMax, OpenAI, Anthropic, or any OpenAI/Anthropic-compatible API based on model ID in the request.
- **OAuth passthrough** — For Anthropic, the client's OAuth Bearer token is forwarded directly. MetaClaw does not manage Anthropic auth; the upstream agent (e.g., Hermes) handles token lifecycle.
- **x-api-key support** — Anthropic-style `x-api-key` header authentication for providers that use it.
- **OpenClaw integration** — OpenClaw is the primary consumer. Agents reference models as `metaclaw/MODEL-ID` to route through MetaClaw.
- **Hermes stays direct** — Hermes (Claude CLI agent) talks directly to Anthropic. It does NOT route through MetaClaw. MetaClaw is for OpenClaw agents only.


## 2. ARCHITECTURE DIAGRAM

```
                        OpenClaw Agents
                    (Aria, Lyra, Pulse, etc.)
                             |
                             | HTTP POST /v1/chat/completions
                             | model: "metaclaw/MiniMax-M2.7-highspeed"
                             |        "metaclaw/claude-opus-4-6"
                             v
                 +---------------------------+
                 |   MetaClaw Proxy Server   |
                 |   localhost:30000         |
                 |                           |
                 |  _resolve_provider()      |
                 |  Match model_id -> provider|
                 |                           |
                 |  _check_auth()            |
                 |  Multi-provider: skip     |
                 |  Passthrough: forward     |
                 |  Bearer: validate         |
                 |                           |
                 |  _forward_to_llm()        |
                 |  Build headers, route     |
                 +--+--------+----------+---+
                    |        |          |
          +---------+   +----+----+   +-+----------+
          |             |         |                 |
          v             v         v                 v
   +-----------+  +-----------+  +-----------+  +----------+
   | MiniMax   |  | OpenAI    |  | Anthropic |  | (others) |
   | api.      |  | api.      |  | api.      |  |          |
   | minimax.  |  | openai.   |  | anthropic.|  |          |
   | io/       |  | com/v1    |  | com       |  |          |
   | anthropic |  |           |  |           |  |          |
   +-----------+  +-----------+  +-----------+  +----------+
   auth: bearer   auth: bearer   auth:          auth: varies
   format:        format:        passthrough
   anthropic      openai         format:
                                 anthropic


   === Direct connections (bypass MetaClaw) ===

   Hermes (Claude CLI)  ----->  Anthropic API (direct, OAuth)
   Katana (Codex CLI)   ----->  OpenAI API (direct, CLI backend)
   Mileena (Codex CLI)  ----->  OpenAI API (direct, CLI backend)
   Gemini CLI agents    ----->  Google API (direct, CLI backend)
```


## 3. CONFIGURATION

### Config file location

```
~/.metaclaw/config.yaml
```

### Full structure with multi-provider routing

```yaml
mode: skills_only

# --- Legacy single-upstream LLM (fallback when model not in providers) ---
llm:
  provider: custom
  api_base: "https://api.anthropic.com"
  api_key: ""
  model_id: "claude-opus-4-6"

# --- Legacy auth settings (used when providers section is absent) ---
llm_auth_passthrough: true
llm_upstream_format: "anthropic"
llm_extra_headers: '{"anthropic-beta": "interleaved-thinking-2025-05-14,..."}'

# --- Proxy ---
proxy:
  port: 30000
  host: "127.0.0.1"

# --- Skills ---
skills:
  enabled: true
  dir: "~/.metaclaw/skills"
  retrieval_mode: "template"
  top_k: 6
  auto_evolve: true
  evolution_every_n_turns: 10

# --- Memory ---
memory:
  enabled: true
  dir: "~/.metaclaw/memory"
  retrieval_mode: "keyword"

# --- Multi-Provider Routing ---
providers:
  minimax:
    api_base: "https://api.minimax.io/anthropic"
    api_key: "${MINIMAX_API_KEY}"
    format: "anthropic"
    extra_headers: '{}'
    auth_mode: "bearer"
    models:
      - "MiniMax-M2.7-highspeed"
      - "MiniMax-M2.5-highspeed"
  openai:
    api_base: "https://api.openai.com/v1"
    api_key: ""
    format: "openai"
    extra_headers: '{}'
    auth_mode: "bearer"
    models:
      - "gpt-5.4"
      - "gpt-5.3-codex"
  anthropic:
    api_base: "https://api.anthropic.com"
    api_key: ""
    format: "anthropic"
    extra_headers: '{"anthropic-beta": "interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14"}'
    auth_mode: "passthrough"
    models:
      - "claude-opus-4-6"
      - "claude-sonnet-4-6"

# --- RL/Scheduler (disabled for proxy-only use) ---
rl:
  enabled: false
scheduler:
  enabled: false
```

### Provider fields

| Field | Type | Description |
|-------|------|-------------|
| `api_base` | string | Base URL of the upstream API (no trailing slash) |
| `api_key` | string | API key, or env var reference like `${MINIMAX_API_KEY}` |
| `format` | string | `"openai"` or `"anthropic"` — determines upstream endpoint and message format |
| `extra_headers` | string | JSON dict of extra HTTP headers to merge (e.g., Anthropic beta headers) |
| `auth_mode` | string | How to authenticate upstream: `"bearer"`, `"passthrough"`, or `"x-api-key"` |
| `models` | list | Model IDs that route to this provider |

### Auth modes

| Mode | Behavior |
|------|----------|
| `bearer` | Sends configured `api_key` as `Authorization: Bearer <key>`. Default mode. |
| `passthrough` | Forwards the client's incoming `Authorization` and `x-api-key` headers to upstream. MetaClaw does not add its own auth. Used for Anthropic where Hermes/OpenClaw manages OAuth tokens. |
| `x-api-key` | Sends configured `api_key` as `x-api-key` header. Also forwards client `Authorization` if present. |

### Environment variable resolution

API keys support `${ENV_VAR}` syntax. At request time, `_resolve_provider()` checks:
```python
if api_key.startswith("${") and api_key.endswith("}"):
    env_var = api_key[2:-1]
    api_key = os.environ.get(env_var, "")
```

Env vars must be set in the launchd plist (for the service) or in the shell (for manual runs).

### Legacy fallback

When a requested model does NOT match any provider in the `providers` section, MetaClaw falls back to:
- `llm.api_base` as the upstream URL
- `llm.api_key` as the bearer token
- `llm_auth_passthrough` to determine passthrough vs bearer
- `llm_upstream_format` for `"openai"` vs `"anthropic"` format
- `llm_extra_headers` for additional headers


## 4. KEY FILES

### metaclaw/config.py

**MetaClawConfig dataclass** — the canonical config object used throughout the codebase.

Key fields for multi-provider routing:
- `providers: str = ""` — JSON string of the providers dict. Populated by `config_store.py` from the YAML `providers:` section. Empty string means legacy mode.
- `llm_api_base`, `llm_api_key`, `llm_model_id` — legacy single-upstream fields
- `llm_auth_passthrough: bool` — legacy passthrough toggle
- `llm_upstream_format: str` — `"openai"` or `"anthropic"` (legacy)
- `llm_extra_headers: str` — JSON dict string (legacy)
- `proxy_port: int = 30000`, `proxy_host: str = "0.0.0.0"`

### metaclaw/config_store.py

**ConfigStore** — reads `~/.metaclaw/config.yaml`, merges with defaults, bridges to MetaClawConfig.

Key logic at line ~191-193:
```python
providers_raw = data.get("providers", {})
providers_json = _json.dumps(providers_raw) if providers_raw else ""
```

The YAML `providers:` dict is serialized to a JSON string and stored in `MetaClawConfig.providers`. This is because the dataclass uses simple types — no nested dicts.

### metaclaw/api_server.py

**MetaClawAPIServer** — FastAPI application. The multi-provider routing lives here.

Key methods:

| Method | Line | Purpose |
|--------|------|---------|
| `_resolve_provider(model_id)` | ~1760 | Matches model_id against each provider's models list. Returns dict with `api_base`, `api_key`, `format`, `extra_headers`, `auth_mode`. Falls back to legacy `llm_*` fields. |
| `_check_auth(authorization, x_api_key)` | ~1135 | In multi-provider mode (providers configured), client auth is optional — skips validation. In passthrough mode, requires Bearer or x-api-key. In bearer mode, validates against configured key. |
| `_forward_to_llm(body, upstream_authorization, upstream_x_api_key)` | ~1812 | Resolves provider, builds auth headers based on auth_mode, merges extra_headers, then dispatches to `_forward_to_openai()` or `_forward_to_anthropic()` based on format. |
| `_forward_to_openai(body, api_base, headers)` | ~1879 | Sends to `/chat/completions`. Strips Tinker-specific fields. Parses inline tool calls from response text. |
| `_forward_to_anthropic(body, api_base, headers)` | ~1929 | Translates OpenAI messages to Anthropic format (system → top-level param, tool messages → tool_result blocks), sends to `/v1/messages`, translates response back to OpenAI format. |

Key endpoints:
- `GET /healthz` — returns `{"ok": true}` (line ~611)
- `POST /v1/chat/completions` — main LLM proxy endpoint (line ~615)

### ~/.metaclaw/config.yaml

Runtime configuration. See Section 3 above for full structure.

### ~/Library/LaunchAgents/ai.metaclaw.proxy.plist

launchd service definition. Runs MetaClaw as a persistent background service.

Key details:
- Binary: `/Users/simonehunt/.hermes/hermes-agent/venv/bin/metaclaw`
- Args: `start --mode skills_only`
- WorkingDirectory: `/Users/simonehunt/.metaclaw`
- Logs: `~/.metaclaw/logs/metaclaw.log` and `metaclaw.err.log`
- KeepAlive: true (auto-restart on crash)
- ThrottleInterval: 5 seconds
- EnvironmentVariables: `HOME`, `MINIMAX_API_KEY`, `PATH`

Service management:
```bash
# Load/start
launchctl load ~/Library/LaunchAgents/ai.metaclaw.proxy.plist

# Unload/stop
launchctl unload ~/Library/LaunchAgents/ai.metaclaw.proxy.plist

# Restart (unload then load)
launchctl unload ~/Library/LaunchAgents/ai.metaclaw.proxy.plist
launchctl load ~/Library/LaunchAgents/ai.metaclaw.proxy.plist

# Check status
launchctl list | grep metaclaw
```


## 5. OPENCLAW INTEGRATION

### How OpenClaw references MetaClaw

OpenClaw's `~/.openclaw/openclaw.json` uses model prefixes to route through MetaClaw. The `metaclaw/` prefix tells OpenClaw to send requests to the MetaClaw proxy.

In the `agents.defaults.models` section:
```json
{
  "metaclaw/claude-opus-4-6": {},
  "metaclaw/MiniMax-M2.7-highspeed": {},
  "metaclaw/MiniMax-M2.5-highspeed": {}
}
```

The default primary model is:
```json
"model": {
  "primary": "metaclaw/claude-opus-4-6",
  "fallbacks": [
    "minimax/MiniMax-M2.7-highspeed",
    "codex-cli/gpt-5.4"
  ]
}
```

### Model ID format

Format: `PROVIDER_PREFIX/MODEL-ID`

- `metaclaw/MODEL-ID` — Routes through MetaClaw proxy at localhost:30000
- `minimax/MODEL-ID` — Routes directly to MiniMax via OpenClaw's own minimax provider config
- `codex-cli/MODEL-ID` — Routes through Codex CLI backend (Katana/Mileena)
- `claude-cli/MODEL-ID` — Routes through Claude CLI backend (Hermes — direct to Anthropic)
- `openrouter-aria/MODEL-ID` — Routes through OpenRouter (direct)

When OpenClaw sends a request with model `metaclaw/MiniMax-M2.7-highspeed`, MetaClaw receives the model ID `MiniMax-M2.7-highspeed` and `_resolve_provider()` matches it to the minimax provider config.

### Which agents route through MetaClaw vs direct

| Route | Agents | Path |
|-------|--------|------|
| Through MetaClaw | Any agent using `metaclaw/*` models | OpenClaw → MetaClaw:30000 → upstream API |
| Direct (MiniMax) | Agents using `minimax/*` models | OpenClaw → MiniMax API directly |
| Direct (CLI) | Katana, Mileena (Codex CLI) | OpenClaw → codex-cli backend → OpenAI |
| Direct (CLI) | Hermes (Claude CLI) | OpenClaw → claude-cli backend → Anthropic |
| Direct (OpenRouter) | Agents using `openrouter-*/*` models | OpenClaw → OpenRouter API |

### Fallback behavior

OpenClaw handles fallbacks at the agent level. If the primary model (`metaclaw/claude-opus-4-6`) fails, OpenClaw tries fallbacks in order. MetaClaw itself does not implement fallback logic — it either routes successfully or returns a 502/503 error.


## 6. ADDING A NEW PROVIDER

### Step-by-step

1. **Edit `~/.metaclaw/config.yaml`** — Add a new entry under `providers:`:

```yaml
providers:
  # ... existing providers ...
  deepseek:
    api_base: "https://api.deepseek.com/v1"
    api_key: "${DEEPSEEK_API_KEY}"
    format: "openai"
    extra_headers: '{}'
    auth_mode: "bearer"
    models:
      - "deepseek-v3.2"
      - "deepseek-r2"
```

2. **Set the environment variable** — Add to the launchd plist:

```xml
<key>DEEPSEEK_API_KEY</key>
<string>sk-xxx</string>
```

Or set the key directly in the YAML (not recommended for secrets):
```yaml
    api_key: "sk-xxx"
```

3. **Restart MetaClaw**:

```bash
launchctl unload ~/Library/LaunchAgents/ai.metaclaw.proxy.plist
launchctl load ~/Library/LaunchAgents/ai.metaclaw.proxy.plist
```

4. **Register in OpenClaw** (optional, for OpenClaw agents to use it):

Add the model to `~/.openclaw/openclaw.json` under `agents.defaults.models`:
```json
"metaclaw/deepseek-v3.2": {}
```

5. **Test**:

```bash
curl -s http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "deepseek-v3.2", "messages": [{"role": "user", "content": "hello"}]}'
```

### Choosing format and auth_mode

| Upstream API style | format | auth_mode |
|-------------------|--------|-----------|
| OpenAI-compatible (`/v1/chat/completions`) | `"openai"` | `"bearer"` |
| Anthropic-compatible (`/v1/messages`) | `"anthropic"` | `"bearer"` or `"passthrough"` |
| Uses x-api-key header | any | `"x-api-key"` |
| Client manages auth (OAuth) | any | `"passthrough"` |


## 7. TROUBLESHOOTING

### Health check

```bash
curl http://localhost:30000/healthz
# Expected: {"ok":true}
```

### Logs

```
~/.metaclaw/logs/metaclaw.log      # stdout
~/.metaclaw/logs/metaclaw.err.log  # stderr
```

Tail logs:
```bash
tail -f ~/.metaclaw/logs/metaclaw.log
```

Provider routing decisions are logged at INFO level:
```
[MetaClaw] Provider routing: model=MiniMax-M2.7-highspeed -> provider=minimax (base=https://api.minimax.io/anthropic, format=anthropic, auth=***)
```

### Common issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Env var not set | `api_key` resolves to empty string, 401 from upstream | Add env var to launchd plist `EnvironmentVariables` section, then reload |
| Auth mode mismatch | 401 or 403 from upstream | Check provider's `auth_mode` — use `"passthrough"` for OAuth providers, `"bearer"` for API key providers |
| Port conflict | MetaClaw fails to start, "address already in use" | Check `lsof -i :30000`, kill conflicting process or change port in config.yaml |
| Model not found in providers | Falls back to legacy `llm_*` config, may hit wrong upstream | Add the model ID to the correct provider's `models` list |
| Wrong format | Garbled response, 400 from upstream | Check `format` matches the upstream API — `"openai"` for OpenAI-compatible, `"anthropic"` for Anthropic Messages API |
| Service not running | Connection refused on localhost:30000 | `launchctl load ~/Library/LaunchAgents/ai.metaclaw.proxy.plist` |
| Stale config after edit | Changes not reflected | Restart: unload + load the launchd plist |
| Extra headers parse error | Warning in logs, headers not applied | Ensure `extra_headers` is valid JSON string (use single quotes around the YAML value) |

### Service status

```bash
# Check if running
launchctl list | grep metaclaw

# Check PID
pgrep -f "metaclaw start"

# Force restart
launchctl kickstart -k gui/$(id -u)/ai.metaclaw.proxy
```

### Manual start (for debugging)

```bash
# Run in foreground with visible output
~/.hermes/hermes-agent/venv/bin/metaclaw start --mode skills_only
```

### Verify provider routing

```bash
# Test MiniMax routing
curl -s http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "MiniMax-M2.7-highspeed", "messages": [{"role": "user", "content": "ping"}]}' \
  | python3 -m json.tool

# Test Anthropic passthrough (needs OAuth token)
curl -s http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <oauth-token>" \
  -d '{"model": "claude-opus-4-6", "messages": [{"role": "user", "content": "ping"}]}' \
  | python3 -m json.tool
```


## 8. REQUEST FLOW (DETAILED)

Step-by-step for a request with model `MiniMax-M2.7-highspeed`:

1. OpenClaw agent sends POST `/v1/chat/completions` to `localhost:30000` with `{"model": "MiniMax-M2.7-highspeed", ...}`

2. FastAPI endpoint handler calls `_check_auth()`:
   - `self.config.providers` is non-empty → client auth is optional → returns immediately

3. Handler calls `_forward_to_llm(body, upstream_authorization, upstream_x_api_key)`

4. `_forward_to_llm()` extracts `model_id = "MiniMax-M2.7-highspeed"` from body

5. Calls `_resolve_provider("MiniMax-M2.7-highspeed")`:
   - Parses `self.config.providers` JSON string
   - Iterates providers: checks minimax.models → `"MiniMax-M2.7-highspeed"` matches
   - Resolves `api_key`: `"${MINIMAX_API_KEY}"` → `os.environ["MINIMAX_API_KEY"]` → actual key
   - Returns: `{api_base: "https://api.minimax.io/anthropic", api_key: "sk-...", format: "anthropic", auth_mode: "bearer", extra_headers: "{}"}`

6. Builds headers:
   - `auth_mode == "bearer"` → `headers["Authorization"] = "Bearer sk-..."`
   - Parses `extra_headers` JSON → empty dict → no merge

7. Checks `format == "anthropic"` → calls `_forward_to_anthropic(body, api_base, headers)`

8. `_forward_to_anthropic()`:
   - Translates OpenAI messages to Anthropic format (system messages → top-level `system` param)
   - POSTs to `https://api.minimax.io/anthropic/v1/messages`
   - Translates Anthropic response back to OpenAI format
   - Returns to client
