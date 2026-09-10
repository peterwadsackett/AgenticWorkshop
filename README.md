# Agentic Workshop

We run the language models on DTU's compute servers. Students use
**OpenCode** on their own computers to chat with the models and work on the exercises.

[Peter and Dimitrios](#1-peter-and-dimitrios--running-the-workshop) ·
[Students](#2-students--connecting-with-opencode) ·
[Download the workshop ZIP](https://github.com/peterwadsackett/AgenticWorkshop/archive/refs/heads/main.zip)

## 1. Peter and Dimitrios — running the workshop

### Connect to the server

```bash
ssh -J dimkan@login.healthtech.dtu.dk dimkan@compute04
cd /home/local/workshop
```

You can connect to compute05 instead. Run the commands below on a compute
node, not the login node. The folder belongs to `dimkan`; Peter needs access
through that account or administrator-arranged permissions.

### Start, stop and check the models

These commands work from either compute node:

| Scope | Start | Stop | Check status |
|---|---|---|---|
| compute04 models | `bash bin/llm start compute04` | `bash bin/llm kill compute04` | `bash bin/llm status compute04` |
| compute05 model | `bash bin/llm start compute05` | `bash bin/llm kill compute05` | `bash bin/llm status compute05` |
| Everything | `bash bin/llm start all` | `bash bin/llm kill all` | `bash bin/llm status all` |

**Before the workshop, run `bash bin/llm start all` and check the readiness summary for both nodes.**

Start commands also start any missing connection services. Already-running
models are kept. If a start fails, read the error before trying again.
`kill` and `stop` mean the same thing: stop our workshop processes safely.
To restart, stop first, wait for success, then start.

> The shared connection service, called the **gateway**, runs on compute04.
> Stopping compute04 disconnects every model, including Qwen 3.8.
> Starting compute05 automatically starts that gateway if needed, but does
> not start compute04's other models.

No tmux is needed, and closing SSH leaves the models running. After a server
reboot, start them again. Only use this setup during the approved workshop
reservation; do not bypass the checks for occupied GPUs.

For just one model, run its command on the node listed below:

```bash
# Example on compute04
bash bin/llm start mistral
bash bin/llm stop mistral

# Example on compute05
bash bin/llm start qwen38
bash bin/llm stop qwen38
```

To preview a command without doing anything, add `--dry-run`.
For more options, run `bash bin/llm --help`.

### Model settings

| Model ID | Node | GPUs | Default context | Server output cap |
|---|---|---|---:|---:|
| `qwen36` | compute04 | 0, 1 | 16384 | 4096 |
| `qwen38` | compute05 | 0, 1 | 32768 | 8192 |
| `mistral` | compute04 | 2 | 16384 | 4096 |
| `gptoss` | compute04 | 3 | 16384 | 8192 |

**Context** is the total space for the conversation and answer, measured in
tokens. It includes previous messages, instructions and tool results.
**Output** is the maximum length of the model's answer, including reasoning.

To change a model's context, stop it first. For example, on compute04:

```bash
bash bin/llm stop mistral
bash bin/llm start mistral --context 32768
```

- This changes one launch only. To save a default, edit that model's
  `context` in `config/models.json` on its node, then stop/start it.
- Update the matching `limit.context` in the student JSON before sharing it.
  Students must restart OpenCode after replacing their configuration.
- Larger contexts need more GPU memory and must be tested. The launcher
  permits up to 65536 for the Qwen models and 32768 for Mistral/GPT-OSS;
  those are allowed settings, not guaranteed working capacities.
- To return Mistral to its default, stop it and start it with
  `--context 16384`. An already-running model ignores a new start request.
- Keep student output at 4096. The server may reduce it further to fit the
  conversation. If the conversation is too long, compact it or start a new chat.

The output caps are set in compute04's `app/gateway.py`, separately from
server context. Changing that code requires tests and a gateway restart.

### Check connections and investigate errors

The launcher waits up to five minutes per node. It checks the gateway and
model health, then sends a tiny test message through each model's public URL.
Each model is marked **READY** only after it returns answer text. Models are
checked independently, so a slow or failed model does not hide the others.

The final summary lists every requested model as **READY** or **NOT READY**,
with a reason for any failure. A failure or timeout makes the command report
an incomplete startup; it does not stop the models that are running. Check
both nodes: compute04 being ready does not mean compute05 also started.

To repeat the checks, run the same start command again. Already-running
models are not reloaded. This tests basic replies, not every possible
OpenCode tool call or a student's network connection.

From a computer with access to the teaching URL:

```bash
curl --fail --max-time 15 https://teaching.healthtech.dtu.dk/workshop/mistral/health
curl --fail --max-time 15 https://teaching.healthtech.dtu.dk/workshop/mistral/models
```

Replace `mistral` with another model ID. Look for `available: true`;
`max_model_len` shows the running context size.

On the model's node:

```bash
bash bin/llm status all
tail -n 60 logs/direct-mistral.log
```

Use `direct-qwen36.log`, `direct-qwen38.log` or `direct-gptoss.log` for
the other models. Connection-service errors are in compute04's
`logs/direct-gateway.log`. Re-running the model's start command restores
missing required services without reloading an already-running model.

To restart only the gateway, on compute04:

```bash
bash bin/llm stop gateway
bash bin/llm start gateway
```

This briefly interrupts all connections but leaves the models loaded.

### Files and ports we maintain

| Location on the cluster | What it is for |
|---|---|
| `bin/llm.py` | Starting, stopping and checking services |
| `app/` | Connection handling and model requests |
| `config/models.json` | Model settings and server addresses |
| `config/model-endpoints.json`, `config/reverse-proxy.json` | Public-connection settings |
| `config/access-policy.json` | Whether students need an API key |
| `models/`, `envs/`, `runtime/` | Models and installed software; leave these in place |
| `logs/`, `run/`, `cache/`, `tmp/` | Logs and working files; do not clear while running |

We use **vLLM 0.19.0** to run the models. Peter's HTTPS server forwards
each model URL to the corresponding HTTP address:

| Model | Internal address |
|---|---|
| qwen36 | `10.57.11.104:28102` |
| qwen38 | `10.57.11.105:28101` |
| mistral | `10.57.11.104:28103` |
| gptoss | `10.57.11.104:28104` |

The shared gateway is on compute04 port 28100. It must reach Qwen 3.8's
private model service on compute05 port 28201. These are not student URLs.

Student API keys are currently disabled. Keep the teaching URLs restricted
to the intended audience. Internal credentials are still required; do not
publish server `config/`, keys or logs. The student `opencode.json`
contains no credentials and is intended for sharing.

To check code changes on compute04:

```bash
source bin/workshop-env.sh
PYTHONPATH=/home/local/workshop envs/manage/bin/python -m unittest discover -s tests -p 'test_*.py'
```

## 2. Students — connecting with OpenCode

You only need **OpenCode and the configuration below**. You do not need a
cluster account, SSH, model downloads or server-start commands.

### Quick start — download, extract and open

1. Install [OpenCode](https://opencode.ai).
2. Open [the workshop repository](https://github.com/peterwadsackett/AgenticWorkshop)
   and choose **Code → Download ZIP**.
3. Extract the ZIP and move the extracted folder wherever you want to work.
   Its default name is `AgenticWorkshop-main`.
4. In **OpenCode Desktop**, open that extracted folder as your project—the
   folder containing both `README.md` and `opencode.json`, not its parent.
   For the terminal version, run `opencode` from inside that folder.
5. Start a new chat. Open the model selector and choose a model under **DTU**.
   In the terminal version, type `/models`.

**No JSON editing, manual merging, API key or Custom provider setup is needed.**
Leave your existing global `opencode.jsonc`, `package.json` and
`node_modules` alone. OpenCode combines the project configuration with your
global settings automatically.

The workshop project shows one **DTU** group containing:

| Model | Context | Maximum output |
|---|---:|---:|
| Qwen 3.6 35B-A3B | 16384 | 4096 |
| Qwen 3.8 27B FP8 | 32768 | 4096 |
| Mistral Nemo 12B | 16384 | 4096 |
| GPT-OSS 20B | 16384 | 4096 |

GPT-OSS is the default model and handles small background tasks such as chat
titles. Only the DTU provider is enabled **inside this project**, so old
workshop entries and other providers do not clutter the model list. Your
global configuration is not changed.

Keep exercise files in this folder. If you downloaded an earlier version,
download and open a fresh copy for the new configuration; keep your existing
exercise work. Fully quit and reopen OpenCode if it still shows the old list.

**Does opening the folder start the models?** No. It connects to models that
Peter and Dimitrios have already started. If a model is unavailable, ask an
organiser. You do not need server access.

See [OpenCode project configuration](https://opencode.ai/docs/config/#per-project).

### Manual connection reference — optional

<details>
<summary>Connection fields for a separate project without the supplied JSON</summary>

The ZIP method above is the workshop setup. These fields are only a fallback
for connecting in another project. The form creates separate providers; it
does not reproduce the single DTU group. Do not add these providers in the
workshop project, where only `dtu` is enabled.

Choose **Custom provider** and use the fields below. Create a separate
provider for each model you want. The connection type is **OpenAI-compatible**.

Use each Base URL exactly as shown, with no additional path at the end.
After entering the provider and model fields, select **Submit**.

The connection form may not offer context/output fields. If it does not,
set those limits in JSON using the instructions after the tables.
The supplied JSON already contains them.
See [OpenCode custom providers](https://opencode.ai/docs/providers/#custom-provider).

#### Qwen 3.6 35B-A3B

| Field in OpenCode | What to enter |
|---|---|
| Provider ID | `dtu-qwen36` |
| Display name | `DTU Qwen 3.6` |
| Base URL | `https://teaching.healthtech.dtu.dk/workshop/qwen36` |
| API key | Leave empty |
| Models → Model ID | `qwen36` |
| Models → Display Name | `Qwen 3.6 35B-A3B` |
| Headers | Leave empty; do not add a header |
| Context limit, in JSON | `16384` |
| Output limit, in JSON | `4096` |

#### Qwen 3.8 27B FP8

| Field in OpenCode | What to enter |
|---|---|
| Provider ID | `dtu-qwen38` |
| Display name | `DTU Qwen 3.8` |
| Base URL | `https://teaching.healthtech.dtu.dk/workshop/qwen38` |
| API key | Leave empty |
| Models → Model ID | `qwen38` |
| Models → Display Name | `Qwen 3.8 27B FP8` |
| Headers | Leave empty; do not add a header |
| Context limit, in JSON | `32768` |
| Output limit, in JSON | `4096` |

#### Mistral Nemo 12B

| Field in OpenCode | What to enter |
|---|---|
| Provider ID | `dtu-mistral` |
| Display name | `DTU Mistral` |
| Base URL | `https://teaching.healthtech.dtu.dk/workshop/mistral` |
| API key | Leave empty |
| Models → Model ID | `mistral` |
| Models → Display Name | `Mistral Nemo 12B` |
| Headers | Leave empty; do not add a header |
| Context limit, in JSON | `16384` |
| Output limit, in JSON | `4096` |

#### GPT-OSS 20B

| Field in OpenCode | What to enter |
|---|---|
| Provider ID | `dtu-gptoss` |
| Display name | `DTU GPT-OSS` |
| Base URL | `https://teaching.healthtech.dtu.dk/workshop/gptoss` |
| API key | Leave empty |
| Models → Model ID | `gptoss` |
| Models → Display Name | `GPT-OSS 20B` |
| Headers | Leave empty; do not add a header |
| Context limit, in JSON | `16384` |
| Output limit, in JSON | `4096` |


</details>

### Changing your context setting

Only change context when an organiser tells you the server setting has changed.
In `opencode.json`, find the model under `provider → dtu → models → model ID`.
For Mistral, the limits look like this:

```json
{
  "name": "Mistral Nemo 12B",
  "limit": {
    "context": 16384,
    "output": 4096
  }
}
```

This is one model entry, not a complete OpenCode configuration. Save the file and
restart OpenCode. Increasing this number does not increase the server's capacity.

### If something does not work

| What you see | What to do |
|---|---|
| DTU models are missing | Check that OpenCode opened the folder containing `opencode.json`, then restart it. |
| Connection failed / 503 | Ask an organiser to check the model's start command. |
| Invalid API key / 401 | The workshop does not currently require a key; check for old provider settings. |
| Not found / 404 | Use a fresh workshop download; ask an organiser to check the URL if it persists. |
| Old groups or Big Pickle still appear | Check that you opened the newly extracted folder, then fully quit and reopen OpenCode. |
| Too many requests / 429 | Wait briefly and try again; the servers are shared. |
| Context too long | Compact the conversation or start a new chat. |

Send Peter or Dimitrios the **model name and exact error message** if you need
help. Do not disable certificate checks or change server settings yourself.

OpenCode may ask to edit files or run commands **on your computer**. Review
those requests before allowing them. Work in the exercise folder.
When comparing models, start a fresh chat: switching models within a chat
can keep the earlier conversation and tool results.
