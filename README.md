# Workshop LLMs — operator guide

**compute04 + compute05 · vLLM 0.19.0 · Updated 9 September 2026**

We host the models; students use OpenCode on their own laptops.
All server files stay inside `/home/local/workshop`.

[Start & stop](#1-start-stop-and-status) · [Models & URLs](#2-models-and-urls) · [Context](#3-changing-context-size) · [OpenCode](#4-opencode-setup) · [Checks](#5-health-logs-and-troubleshooting) · [Review guide](#6-what-runs-where)

> **Important:** compute04 hosts the shared gateway for every model.
> Killing compute04 also makes Qwen 3.8's public URL unavailable, even when
> Qwen 3.8 is still running on compute05. Node-specific commands do not
> automatically start or stop the other node.

## 1. Start, stop and status

### Connect

From your laptop, connect to either node through the login host:

```bash
ssh -J dimkan@login.healthtech.dtu.dk dimkan@compute04
cd /home/local/workshop
```

Use compute05 instead to work there. The login host is only an SSH jump;
do not run the models on it. Commands below run on a compute node.

The workshop belongs to `dimkan` and has owner-only permissions.
An administrator using another account must arrange access or run as the
workshop owner. Cross-node commands use noninteractive SSH as `dimkan`;
they do not disable SSH host-key checking.

### Command reference — works from either compute node

| What to control | Start | Kill / stop | Status |
|---|---|---|---|
| **Only compute04** | `bash bin/llm start compute04` | `bash bin/llm kill compute04` | `bash bin/llm status compute04` |
| **Only compute05** | `bash bin/llm start compute05` | `bash bin/llm kill compute05` | `bash bin/llm status compute05` |
| **Both nodes** | `bash bin/llm start all` | `bash bin/llm kill all` | `bash bin/llm status all` |

`kill` and `stop` mean the same thing: gracefully stop verified workshop
processes. They do not reboot the computer, force-kill unrelated jobs or touch Atlas.

To restart a node, run its kill command, wait for successful completion, then
run its start command. Do not start again if stopping reports an error.

For both nodes, stop order is compute05 → compute04; start order is
compute04 → compute05. An SSH preflight failure changes neither node.
Later failures are reported; always inspect status after an incomplete operation.

### One model, local-only control, and previews

Individual-model commands must run on the model's own node:

```bash
# On compute04; replace mistral with qwen36 or gptoss if needed
bash bin/llm stop mistral
bash bin/llm start mistral

# On compute05
bash bin/llm stop qwen38
bash bin/llm start qwen38
```

The matching model endpoint is included automatically.

```bash
# Preview without changing anything
bash bin/llm kill compute04 --dry-run
bash bin/llm start all --dry-run

# Affect only the node you are currently on
bash bin/llm stop all --local

# Show all available options
bash bin/llm --help
```

No tmux is needed. Processes run in the background after SSH closes, but do
**not** restart automatically after reboot. Use direct execution only during
the administrator-approved workshop reservation. GPU/Slurm allocation checks
remain enabled. A started PID is not proof that model loading has finished.

## 2. Models and URLs

| Model ID | Model | Node / GPUs | Default context | Server output cap |
|---|---|---|---:|---:|
| `qwen36` | Qwen 3.6 35B-A3B | compute04 / 0, 1 | 16384 | 4096 |
| `mistral` | Mistral Nemo 12B | compute04 / 2 | 16384 | 4096 |
| `gptoss` | GPT-OSS 20B | compute04 / 3 | 16384 | 8192 |
| `qwen38` | Qwen 3.8 27B FP8 | compute05 / 0, 1 | 32768 | 8192 |

Use these base URLs exactly; **do not append `/v1`**:

| Model ID | OpenCode base URL | Administrator's HTTP upstream |
|---|---|---|
| `qwen36` | `https://teaching.healthtech.dtu.dk/workshop/qwen36` | `10.57.11.104:28102` |
| `mistral` | `https://teaching.healthtech.dtu.dk/workshop/mistral` | `10.57.11.104:28103` |
| `gptoss` | `https://teaching.healthtech.dtu.dk/workshop/gptoss` | `10.57.11.104:28104` |
| `qwen38` | `https://teaching.healthtech.dtu.dk/workshop/qwen38` | `10.57.11.105:28101` |

The spelling is **gptoss**, not qptoss. HTTPS certificates are handled by the
administrator's proxy. Student API-key and extra-header fields are empty during
keyless testing. Anyone who can reach these public routes can use them;
the administrator should restrict exposure to the intended audience.

## 3. Changing context size

**Context is the total budget for input plus output.** Input includes chat
history, system instructions, tool definitions and tool results—not just the
latest message.

### Change one launch

Stop the model first; starting an already-running model does not change it.

```bash
# Example: test a 32K Mistral context, on compute04
bash bin/llm stop mistral
bash bin/llm start mistral --context 32768
```

Wait until it is ready, then update OpenCode's context setting to match.
If it fails to load, stop it and return to the default:

```bash
bash bin/llm stop mistral
bash bin/llm start mistral --context 16384
```

| Model | Default | Current launcher upper bound |
|---|---:|---:|
| qwen36 | 16384 | 65536 |
| mistral | 16384 | 32768 |
| gptoss | 16384 | 32768 |
| qwen38 | 32768 | 65536 |

These upper bounds are configured guardrails, **not tested capacity guarantees**.
Larger contexts use more GPU memory and can reduce concurrency.
The launcher minimum is 2048. `--context` applies to one local model, not to
`all`, `compute04`, `compute05`, the gateway or an endpoint.

### Make a default permanent

On the node hosting that model, edit `config/models.json`:

- `context`: the default used at the next normal start.
- `max_context`: the launcher's allowed upper bound, not a GPU-capacity promise.

Preserve the other profile fields. Stop and start that model to apply the
change. A command-line `--context` override is not saved to this file.

### Output budgets and sampling

The client should request 4096 output tokens by default. The gateway enforces
the model caps in the table, even if a client requests more. After a
pre-generation context rejection, it can count the input and retry once with
a smaller output allowance. If input itself is too long, compact the chat or
start a new one; the gateway does not silently truncate history.

Output policy lives in compute04's `app/gateway.py`, in the `max_tokens` /
`max_completion_tokens` handling. Changing that code requires tests and a
gateway restart; it is separate from a model's server context.

Temperature is a request-time sampling setting, not a context setting.
The `temperature` entries in `config/models.json` are not applied by the
current launcher; editing them alone does not change client requests.

## 4. OpenCode setup

Create one custom OpenAI-compatible provider per model, using its base URL and
exact model ID above. Leave the API key empty. Use
`@ai-sdk/openai-compatible` for JSON configuration.

Example project `opencode.json` for Mistral at its **default** context:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "dtu-mistral": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "DTU Mistral",
      "options": {
        "baseURL": "https://teaching.healthtech.dtu.dk/workshop/mistral"
      },
      "models": {
        "mistral": {
          "name": "Mistral Nemo 12B",
          "limit": { "context": 16384, "output": 4096 }
        }
      }
    }
  },
  "permission": { "edit": "ask", "bash": "ask" }
}
```

Edit `provider → dtu-mistral → models → mistral → limit` to match the
running server. This example configures only Mistral; merge it into an existing
configuration rather than overwriting other providers.
See the [OpenCode custom-provider documentation](https://opencode.ai/docs/providers/#custom-provider).

For user-wide settings, use `~/.config/opencode/opencode.json` on the
student's laptop; project settings can override matching global settings.
Restart OpenCode after editing.
See [configuration locations](https://opencode.ai/docs/config/#locations).

Use a dedicated exercise folder and review proposed edits/commands.
Switching models in an existing chat can retain its conversation and tool
history; use a fresh chat for clean model comparisons.

## 5. Health, logs and troubleshooting

### Check readiness

Run from a machine with access to the teaching URL. Replace `mistral` with
the desired model ID:

```bash
curl --fail --max-time 15 https://teaching.healthtech.dtu.dk/workshop/mistral/health
curl --fail --max-time 15 https://teaching.healthtech.dtu.dk/workshop/mistral/models
```

Wait for `available: true`. The model listing reports the live
`max_model_len`; use that to confirm a context change.

On the relevant compute node:

```bash
cd /home/local/workshop
bash bin/llm status all
tail -n 60 logs/direct-mistral.log
nvidia-smi
```

For Qwen 3.8 use compute05's `logs/direct-qwen38.log`; gateway errors are
in compute04's `logs/direct-gateway.log`. `logs/requests.jsonl` records
request status/timing without storing prompts or completions.

| Problem | Check / action |
|---|---|
| Process started but API unavailable | Model may still be loading; inspect logs and `/health`. |
| Qwen 3.8 runs but its URL fails | Check compute04's shared gateway. |
| HTTP 404 | Check base URL/model spelling; GPT-OSS uses `gptoss`. |
| HTTP 401 | Unexpected in keyless mode; check route/access policy. |
| HTTP 429 | Shared rate/concurrency limit; retry with backoff. |
| HTTP 503 / unavailable | Check model and gateway status; inspect logs. |
| Context error | Compact/start a fresh chat; align client/server limits. |
| GPUs occupied or Slurm allocation present | Do not bypass the guard or kill others' jobs; ask the administrator. |
| SSH preflight failed | Check inter-node access; use an explicit local node only if that is what you intend. |

### Restart only the gateway

On compute04:

```bash
bash bin/llm stop gateway
bash bin/llm start gateway
```

This leaves GPU models loaded but briefly interrupts API access for all models.

### Run tests without changing model state

On compute04:

```bash
cd /home/local/workshop
source bin/workshop-env.sh
PYTHONPATH=/home/local/workshop envs/manage/bin/python -m unittest discover -s tests -p 'test_*.py'
```

On compute05, run the current launcher tests with
`PYTHONPATH=/home/local/workshop envs/manage/bin/python -m unittest discover -s tests -p test_launcher.py`
after sourcing `bin/workshop-env.sh`. Its older gateway tests are not the
current compute04 gateway suite.

## 6. What runs where

```text
Student's OpenCode
  → DTU HTTPS proxy
  → per-model endpoint on compute04 or compute05
  → shared gateway on compute04
  → vLLM backend on the model's node
```

OpenCode executes permitted tools on the student's laptop.
The model server supplies responses; it does not execute those shell commands.

| File / folder | Purpose |
|---|---|
| `bin/llm`, `bin/llm.py` | Normal operator entry point, node routing and verified process control |
| `bin/workshop-env.sh` | Keeps Python, temporary files and caches inside workshop |
| `app/model_endpoint.py` | Model-specific URL handling and source restrictions |
| `app/pilot_proxy.py` | Active compute04 gateway entry point; historical filename |
| `app/gateway.py` | Shared routing, credentials, quotas, output limits and streaming |
| `app/request_compat.py` | Context-error handling and Mistral history conversion |
| `config/models.json` | Model profiles, context defaults, backend addresses and vLLM arguments |
| `config/model-endpoints.json`, `config/reverse-proxy.json` | Endpoint and proxy rules |
| `config/access-policy.json` | Student-key requirement; currently false on compute04 |
| `models/`, `envs/`, `runtime/` | Weights, installed packages and Python; do not move/delete |
| `run/`, `logs/`, `cache/`, `tmp/`, `data/` | Process state, diagnostics and runtime data |
| `tests/`, `manifests/` | Regression tests, pinned downloads and version records |

Compute04 is the canonical gateway. Similarly named old files on compute05
are not extra active gateways; use this launcher, not old pilot/workshopctl scripts.

Private backend keys remain necessary despite student keyless mode.
Do not publish `config/`, keys, certificates, logs, model files or environments.
The proxy-source allowlist includes `10.57.3.3` and the two compute nodes.
Internal traffic needs compute05 → compute04 TCP 28100 and
compute04 → compute05 TCP 28201. Firewall/certificate changes belong to the administrator.
