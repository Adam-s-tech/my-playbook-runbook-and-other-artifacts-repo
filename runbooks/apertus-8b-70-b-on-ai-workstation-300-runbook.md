# Apertus v1.5 Runbook — CORSAIR AI WORKSTATION 300

## Purpose

This runbook documents the tested procedure for downloading, serving, validating, and troubleshooting the text-only GGUF builds of **Apertus v1.5 8B** and **Apertus v1.5 70B** on the CORSAIR AI WORKSTATION 300.

It is intentionally practical: the goal is to get from a fresh Ubuntu workstation to a verified local OpenAI-compatible endpoint, not to audition for an interpretive dance called “ROCm Error Messages.”

## Validated system

| Component | Confirmed configuration |
|---|---|
| Host | CORSAIR AI WORKSTATION 300 |
| OS | Ubuntu 26.04 |
| CPU / APU | AMD Ryzen AI Max+ 395 |
| GPU | AMD Radeon 8060S integrated graphics |
| System memory | 128 GB LPDDR5X unified memory |
| ROCm architecture | `gfx1151` |
| llama.cpp launcher | `llama` from `https://llama.app/install.sh` |
| Installed llama build during validation | `0.4.0-dev`, build `10819`, commit `6a1a922d2` |
| ROCm device listed by llama | `ROCm0: AMD Radeon Graphics (98304 MiB, ...)` |

## Validated models

| Model | GGUF source | Local file | Quantization | Endpoint | Measured decode speed |
|---|---|---|---|---|---:|
| Apertus v1.5 8B text | `Colby/apertus-v1.5-8b-text-Q4_K_M-GGUF` | `~/models/apertus/v1.5/8b-text/apertus-v1.5-8b-text-q4_k_m.gguf` | Q4_K_M | `127.0.0.1:8080` | ~38.7–39.3 tokens/sec |
| Apertus v1.5 70B text | `katya228/Apertus-v1.5-70B-text-GGUF` | `~/models/apertus/v1.5/70b-text/apertus-70b-Q4_K_M.gguf` | Q4_K_M | `127.0.0.1:8081` | ~4.6–4.8 tokens/sec |

> These are **community text-only GGUF conversions**. They do not provide the official Apertus v1.5 multimodal image/audio capabilities. Treat the 70B conversion as an evaluation model and verify behavior for your own tasks.

---

## 1. Preconditions

### 1.1 Update the host

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

### 1.2 Install basic utilities

```bash
sudo apt install -y \
  curl wget git unzip zstd \
  python3 python3-venv python3-pip \
  build-essential cmake pciutils mesa-utils
```

### 1.3 Enable GPU device access

```bash
sudo usermod -aG render,video "$USER"
```

Log out and back in after changing groups, then confirm:

```bash
groups
```

The output should include `render` and `video`.

### 1.4 Confirm ROCm sees the APU GPU

```bash
rocminfo | grep -E 'Name:|gfx'
```

Expected evidence includes:

```text
Name:                    gfx1151
```

This verifies the Radeon 8060S is visible to ROCm.

---

## 2. Install llama.cpp launcher

Install the current llama.cpp single-binary launcher:

```bash
curl -LsSf https://llama.app/install.sh | sh
```

The installer probes CUDA, then ROCm, then Vulkan, then CPU on Linux. On this host, ROCm is the desired backend.

Validate the installation:

```bash
which llama
llama version
llama --help | head -n 25
```

Expected executable location:

```text
/home/tim-dickey/.local/bin/llama
```

### 2.1 Confirm llama sees ROCm

```bash
llama serve --list-devices
```

Validated output format:

```text
Available devices:
  ROCm0: AMD Radeon Graphics (98304 MiB, ... MiB free)
```

`ROCm0` is used explicitly in the Apertus launch commands.

---

## 3. Understand model locations

Do not store personal model weights in `~/llama.cpp/models/` merely because it has a `models` name. That directory commonly contains llama.cpp vocabulary and tokenizer-support GGUF files such as `ggml-vocab-*.gguf`, not user model weights.

Use this personal model library instead:

```text
~/models/
├── apertus/
│   └── v1.5/
│       ├── 8b-text/
│       │   └── apertus-v1.5-8b-text-q4_k_m.gguf
│       └── 70b-text/
│           └── apertus-70b-Q4_K_M.gguf
└── Qwen3.8-27B/
```

The `llama` launcher can run cached preset models in router mode, but the Apertus runbooks below use explicit `--model` paths. This is more reproducible and avoids turning local storage into an archaeological excavation.

---

## 4. Install and use Hugging Face CLI

The modern Hugging Face command is `hf`, not `huggingface-cli` or `hf-cli`.

Check it:

```bash
hf --help
```

Authenticate when downloading gated official models:

```bash
hf auth login
hf auth whoami
```

For the community GGUF repositories used here, a token may not be required. Avoid placing tokens directly in commands, shell history, or files.

Search for models using:

```bash
hf models ls --search "Apertus-v1.5-8B" --no-truncate
hf models ls --search "Apertus-v1.5-70B text GGUF" --no-truncate
```

---

## 5. Apertus v1.5 8B setup

### 5.1 Download the 8B text GGUF

Create the destination directory:

```bash
mkdir -p ~/models/apertus/v1.5/8b-text
```

Download the validated 8B Q4_K_M model:

```bash
hf download \
  Colby/apertus-v1.5-8b-text-Q4_K_M-GGUF \
  apertus-v1.5-8b-text-q4_k_m.gguf \
  --local-dir ~/models/apertus/v1.5/8b-text
```

Verify it:

```bash
ls -lh ~/models/apertus/v1.5/8b-text/
```

Validated model file size was approximately 5.1 GB.

### 5.2 Check that port 8080 is free

```bash
sudo ss -ltnp 'sport = :8080'
```

If a process is listening, inspect it before stopping it:

```bash
ps -fp <PID>
```

To stop an old llama server only after confirming it is safe:

```bash
kill <PID>
sleep 2
sudo ss -ltnp 'sport = :8080'
```

### 5.3 Launch 8B

```bash
llama serve \
  --model ~/models/apertus/v1.5/8b-text/apertus-v1.5-8b-text-q4_k_m.gguf \
  --ctx-size 8192 \
  --gpu-layers all \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --host 127.0.0.1 \
  --port 8080 \
  --alias apertus-v1.5-8b
```

Notes:

- `--gpu-layers all` requests maximal practical GPU offload.
- `--ctx-size 8192` is a conservative, practical starting context rather than the model’s 262K training context.
- Q8 K/V cache lowers cache memory compared with default FP16.
- `127.0.0.1` keeps the endpoint local to the workstation.

### 5.4 Validate 8B server availability

In another terminal:

```bash
curl http://127.0.0.1:8080/v1/models
```

Expected model identifier:

```text
apertus-v1.5-8b
```

### 5.5 Validate 8B inference

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "apertus-v1.5-8b",
    "messages": [
      {
        "role": "user",
        "content": "In three concise sentences, explain what unified memory is."
      }
    ],
    "temperature": 0.2,
    "max_tokens": 180
  }'
```

Validated results:

- Model metadata reported approximately 8.05B parameters.
- Active context was 8,192 tokens.
- Short and long tests generated about 38.7–39.3 tokens/sec.
- A 1,100-token output sustained about 38.66 tokens/sec.

### 5.6 8B caution

A warning may appear:

```text
special_eos_id is not in special_eog_ids - the tokenizer config may be incorrect
```

The model still completed successfully in testing. Monitor for failure to stop, visible control tokens, malformed agent output, or broken role handling. If those appear, test an alternative conversion or official runtime.

Stop the server with `Ctrl+C` when finished. Do not load an 8B and 70B model simultaneously unless deliberately benchmarking contention.

---

## 6. Apertus v1.5 70B setup

### 6.1 Download the 70B text GGUF

Create the directory:

```bash
mkdir -p ~/models/apertus/v1.5/70b-text
```

Download the validated model:

```bash
hf download \
  katya228/Apertus-v1.5-70B-text-GGUF \
  apertus-70b-Q4_K_M.gguf \
  --local-dir ~/models/apertus/v1.5/70b-text
```

Verify the model and storage space:

```bash
ls -lh ~/models/apertus/v1.5/70b-text/
df -h ~
```

Validated download details:

- Download size: 43.7 GB.
- Local file reporting: about 41 GiB.
- Example filesystem capacity after download: about 1.4 TB free.

### 6.2 Use a separate port

This workstation had a router-mode llama service on port 8080. The explicit 70B model was served on port 8081 to avoid disrupting it.

Check the port if needed:

```bash
sudo ss -ltnp 'sport = :8081'
```

### 6.3 Launch 70B

Use this **normal-chat** command:

```bash
llama serve \
  --model ~/models/apertus/v1.5/70b-text/apertus-70b-Q4_K_M.gguf \
  --device ROCm0 \
  --ctx-size 4096 \
  --gpu-layers all \
  --flash-attn on \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --override-kv tokenizer.ggml.eos_token_id=int:68 \
  --fit on \
  --fit-target 8192 \
  --host 127.0.0.1 \
  --port 8081 \
  --alias apertus-v1.5-70b
```

Important meanings:

| Option | Reason |
|---|---|
| `--device ROCm0` | Selects the Radeon 8060S as enumerated by llama.cpp |
| `--ctx-size 4096` | Conservative initial context for a 70B model in unified memory |
| `--gpu-layers all` | Requests full practical GPU offload |
| `--flash-attn on` | Enables Flash Attention where supported |
| `--cache-type-k/v q8_0` | Reduces KV-cache memory pressure |
| `--override-kv tokenizer.ggml.eos_token_id=int:68` | Fixes end-of-answer behavior for this community conversion |
| `--fit on --fit-target 8192` | Lets llama.cpp fit model settings while reserving an 8 GiB device-memory margin |
| `--host 127.0.0.1` | Keeps the service private during commissioning |
| `--port 8081` | Avoids collision with a service on 8080 |

### 6.4 Do not add `-sp` for normal chat

The 70B converter’s instructions may mention `-sp` / `--special`. It makes special tokens visible. In testing, this caused a normal answer to include:

```text
<|assistant_end|>
```

For ordinary chat/API use, **omit `-sp`**. Retain the EOS override in the server launch command.

Also, do not type this by itself:

```bash
--override-kv tokenizer.ggml.eos_token_id=int:68
```

It is an option to `llama serve`, not a standalone shell command.

### 6.5 Validate 70B server

```bash
curl http://127.0.0.1:8081/v1/models
```

Expected identifier:

```text
apertus-v1.5-70b
```

Validated model metadata:

| Item | Value |
|---|---:|
| Parameters | 70,599,864,384 (~70.6B) |
| GGUF model size | 43,713,675,520 bytes |
| Quantization | Q4_K_M |
| Active context | 4,096 tokens |
| Training context | 262,144 tokens |

### 6.6 Validate correct stopping behavior

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "apertus-v1.5-70b",
    "messages": [
      {
        "role": "system",
        "content": "You are a concise and accurate assistant."
      },
      {
        "role": "user",
        "content": "What is the capital of Switzerland? Answer in one sentence."
      }
    ],
    "temperature": 0.2,
    "max_tokens": 80
  }'
```

Validated expected behavior:

```json
{
  "finish_reason": "stop",
  "message": {
    "content": "The capital of Switzerland is Bern."
  }
}
```

The normal-chat configuration should not display `<|assistant_end|>`.

### 6.7 Benchmark 70B

Run a sustained real-world request:

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "apertus-v1.5-70b",
    "messages": [
      {
        "role": "user",
        "content": "Explain, in about 600 words with clear headings, why quantization makes large language models practical to run locally. Include one example involving a 70B model."
      }
    ],
    "temperature": 0.3,
    "max_tokens": 1200
  }'
```

Validated 70B performance:

| Metric | Observed result |
|---|---:|
| Short-query prompt processing | 43.38 tokens/sec |
| Short-query generation | 4.83 tokens/sec |
| Long-query prompt processing | 107.34 tokens/sec for a 99-token prompt |
| Long-query sustained generation | 4.61 tokens/sec for 1,200 generated tokens |
| 1,200-token generation time | ~260 seconds |
| Long-run finish | Natural completion, `truncated = 0` |

Practical timing at 4.61 tokens/sec:

- 100 generated tokens: roughly 22 seconds.
- 500 generated tokens: roughly 1 minute 49 seconds.
- 1,000 generated tokens: roughly 3 minutes 37 seconds.

---

## 7. Monitor GPU and memory activity

### 7.1 llama.cpp device report

```bash
llama serve --list-devices
```

This is the most useful planning view for llama.cpp. On this system, it reported a 98,304 MiB ROCm device-memory pool.

### 7.2 AMD monitoring tools

```bash
rocm-smi
```

or:

```bash
amd-smi monitor
```

For periodic monitoring:

```bash
watch -n 0.5 amd-smi monitor
```

Run `watch` in a dedicated terminal. Do not type curl commands in the same terminal, or screen redraws can make the prompt look scrambled.

### 7.3 Unified-memory caveat

On this APU-style unified-memory design, `amd-smi` may report a smaller apparent VRAM pool, such as `4.4 / 64.0 GB`, even while llama.cpp has loaded the 43.7 GB 70B GGUF and identifies a 98,304 MiB ROCm device. Treat actual llama.cpp loading behavior, the `--list-devices` output, and inference performance as stronger evidence than a single narrow monitoring counter.

---

## 8. Troubleshooting

### 8.1 `couldn't bind HTTP server socket`

Cause: The selected port is already in use.

Check it:

```bash
sudo ss -ltnp 'sport = :8080'
sudo ss -ltnp 'sport = :8081'
```

Inspect the process:

```bash
ps -fp <PID>
```

Either stop an unwanted old server:

```bash
kill <PID>
```

or use a different local port.

### 8.2 `failed to open GGUF file ... No such file or directory`

Cause: The directory, filename, case, or path is incorrect, or the model has not yet downloaded.

Find candidate files:

```bash
find ~/models -type f -iname "*apertus*70*.gguf" -printf '%f\t%p\t%k KB\n' | sort
find ~/models -type f -iname "*apertus*8*.gguf" -printf '%f\t%p\t%k KB\n' | sort
```

Use the exact returned full path with `--model`. Linux paths are case-sensitive.

### 8.3 Model starts but emits `<|assistant_end|>`

Cause: `-sp` / `--special` was enabled.

Fix: Stop the server and relaunch without `-sp`. Keep:

```bash
--override-kv tokenizer.ggml.eos_token_id=int:68
```

### 8.4 70B runs out of memory

First stop other heavyweight local models and GPU workloads. Ensure the 8B server is stopped if it was manually launched.

Then retry with a smaller context and larger margin:

```bash
llama serve \
  --model ~/models/apertus/v1.5/70b-text/apertus-70b-Q4_K_M.gguf \
  --device ROCm0 \
  --ctx-size 2048 \
  --gpu-layers all \
  --flash-attn on \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --override-kv tokenizer.ggml.eos_token_id=int:68 \
  --fit on \
  --fit-target 12288 \
  --host 127.0.0.1 \
  --port 8081 \
  --alias apertus-v1.5-70b
```

Do not remove the EOS override as a memory workaround; it is a correctness setting.

### 8.5 `--override-kv: command not found`

Cause: A llama command-line option was pasted directly into Bash.

Fix: Include it as part of the full `llama serve` command. Do not execute it alone.

### 8.6 `hf-cli` not found or Hugging Face warning

Cause: The old `huggingface-cli` command was deprecated. `hf-cli` is not the replacement.

Use:

```bash
hf --help
hf models ls --search "Apertus-v1.5-70B text GGUF" --no-truncate
```

---

## 9. Operational guidance

### 9.1 Model roles

| Model | Use it for | Avoid using it for |
|---|---|---|
| Apertus 8B | Interactive chat, routine coding, lightweight RAG, fast agent experiments, iterative work | The hardest multi-document analysis where quality matters more than time |
| Apertus 70B | Deep synthesis, complex planning, difficult technical review, deliberate long-form analysis | Rapid back-and-forth conversation, latency-sensitive automation, large concurrent workloads |

### 9.2 Network and security posture

During commissioning, bind services to loopback only:

```text
127.0.0.1
```

Do not bind to `0.0.0.0` or expose either service to the LAN/internet merely because the API works locally. The launcher logs warn that its default CORS behavior allows all origins and no API key is set. If you later expose a service beyond localhost, put it behind deliberate authentication, access controls, and a network policy.

### 9.3 Avoid unnecessary concurrent loads

The AI Workstation 300 has a large unified-memory pool, but 70B Q4 inference is a heavyweight workload. Use one manually loaded large model at a time unless intentionally testing contention.

### 9.4 Suggested endpoint convention

| Endpoint | Model | Role |
|---|---|---|
| `http://127.0.0.1:8080` | Apertus v1.5 8B | Fast local endpoint |
| `http://127.0.0.1:8081` | Apertus v1.5 70B | Deliberate high-quality endpoint |

---

## 10. Validation checklist

### 8B

- [ ] `rocminfo` reports `gfx1151`
- [ ] `llama serve --list-devices` lists `ROCm0`
- [ ] 8B GGUF exists in `~/models/apertus/v1.5/8b-text/`
- [ ] `llama serve` starts and listens on port 8080
- [ ] `/v1/models` shows `apertus-v1.5-8b`
- [ ] `/v1/chat/completions` returns coherent content
- [ ] Decode speed is approximately 39 tokens/sec

### 70B

- [ ] 70B GGUF exists in `~/models/apertus/v1.5/70b-text/`
- [ ] Server starts with `--device ROCm0` on port 8081
- [ ] `/v1/models` shows `apertus-v1.5-70b`
- [ ] Switzerland test returns “Bern” with `finish_reason: "stop"`
- [ ] Raw `<|assistant_end|>` token does not appear in normal-chat output
- [ ] Long request reaches roughly 4.6 tokens/sec decode speed
- [ ] Model is bound to `127.0.0.1`, not unintentionally exposed

## Bottom line

The CORSAIR AI WORKSTATION 300 has been validated as a capable local host for both Apertus v1.5 text-only GGUF models:

- **8B Q4_K_M:** fast and genuinely interactive at about 39 tokens/sec.
- **70B Q4_K_M:** stable and usable as a quality-focused local endpoint at about 4.6–4.8 tokens/sec.

That is a practical two-tier local-model setup: use 8B for the conversational sprint, and invite 70B when the task deserves the slow, thoughtful Swiss train ride.