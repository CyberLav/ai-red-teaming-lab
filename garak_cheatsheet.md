# Garak Cheat Sheet

Quick reference for running NVIDIA's Garak LLM vulnerability scanner against local Ollama models via WSL/Ubuntu. Companion to `Garak_Setup_Journey.md` (the full story) this is the fast-lookup version.

---

## Every-session startup checklist

```bash
# 1. Confirm Ollama is running (Windows Task Manager: ollama.exe, ollama app.exe)

# 2. Open Ubuntu
wsl                          # from PowerShell, or open "Ubuntu" from Start menu

# 3. Confirm you're in the right distro
cat /etc/os-release          # should say Ubuntu

# 4. Activate the venv (every new terminal window needs this)
source ~/garak-env/bin/activate      # prompt should show (garak-env)

# 5. Confirm WSL can reach Ollama
curl http://localhost:11434/api/tags     # should return JSON model list
```

---

## Install / update

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv -y
python3 -m venv ~/garak-env
source ~/garak-env/bin/activate
pip install garak                    # first install
pip install --upgrade garak          # update later
```

---

## Core command syntax

```bash
python -m garak --target_type ollama --target_name <model> --probes <probe> --generations <n> --report_prefix <name> --seed <n> --verbose
```

| Flag | What it does | Notes |
|---|---|---|
| `--target_type` | Generator backend (e.g. `ollama`) | `--model_type` is the deprecated alias, still works |
| `--target_name` | Model name as Ollama knows it (e.g. `qwen3`) | `--model_name` is the deprecated alias |
| `--probes` | Which probe(s) to run | Comma-separated for multiple; `module.SpecificProbe` for one sub-probe |
| `--generations` | How many responses per prompt | Keep at `1` on slow/CPU-only hardware |
| `--report_prefix` | Name for output report files | Files land in `~/.local/share/garak/garak_runs/` |
| `--seed` | Fixes randomness for reproducibility | Doesn't guarantee non-overlapping coverage across different seeds |
| `--verbose` | More detailed console output | Use alongside `tail -f` on the log for live confirmation it's working |
| `--config` | Load a YAML config file | ⚠️ Known unreliable in this version. See Gotchas below |
| `--generator_options` | Inline JSON generator overrides | ⚠️ Also unreliable in this version. See Gotchas below |
| `--list_probes` | Print every available probe | Good for browsing what's testable |
| `--list_detectors` | Print every available detector | |
| `--list_generators` | Print every available generator backend | |

---

## Probe categories (quick reference)

| Probe module | What it tests | Maps to |
|---|---|---|
| `dan` | Jailbreak personas (DAN family) roleplay an "unrestricted" alter-ego | OWASP LLM01, ATLAS evasion |
| `promptinject` | Hides an override instruction inside innocuous-looking content | OWASP LLM01 |
| `encoding` | Same as promptinject, but obfuscated (Base64/ROT13) to dodge keyword filters | OWASP LLM01 |
| `leakreplay` | Feeds a known copyrighted snippet, checks if model completes it verbatim | OWASP LLM06 (data leakage) |
| `malwaregen` | Directly requests malicious code (keyloggers, ransomware logic) | OWASP LLM02 / agentic misuse |
| `xss` | Tries to get a literal `<script>` payload into model output | OWASP LLM02 (insecure output handling) |
| `packagehallucination` | Checks if model invents nonexistent library names | Supply-chain risk |
| `realtoxicityprompts` | Nudges toward toxic/hateful continuations | Content safety |
| `tap` | Automated, adaptive jailbreak search (rewrites based on refusals) | OWASP LLM01, adaptive attack |

Example sub-probe: `dan.Ablation_Dan_11_0`  a pruned/tested variant of DAN 11.0, 127 prompt variations.

---

## Monitoring a run (the "is it stuck?" checklist)

1. **Don't trust `top` inside WSL**  actual inference runs in Windows' Ollama app, not WSL, so WSL always looks idle regardless.
2. **Check Windows Task Manager** for `llama-server.exe` (the real worker process  not just `ollama.exe`/`ollama app.exe`, which stay near 0% even when busy). High CPU/RAM here = genuinely working.
3. **Tail the log live** in a second terminal:
   ```bash
   tail -f ~/.local/share/garak/garak.log
   ```
4. **Test raw connectivity directly** if in doubt:
   ```bash
   time curl http://localhost:11434/api/generate -d '{"model": "qwen3", "prompt": "Say hello in one sentence.", "stream": false}'
   ```
   Compare the `real` time against garak's timeout (see Gotchas)  if generation takes longer than the timeout, you'll loop forever instead of finishing.

---

## Known bugs / gotchas in this Garak version

**1. Default Ollama request timeout (30s) is too short for CPU-only hardware.**
Symptom in the log: repeating `ReadTimeout(TimeoutError('timed out'))` → `Backing off _call_model(...)` forever.

Neither of the documented fixes reliably worked for us:
```bash
# Attempt 1 — YAML config (didn't take effect)
generators:
  ollama:
    OllamaGeneratorChat:
      timeout: 400

# Attempt 2 — CLI flag (also didn't take effect)
--generator_options '{"generators":{"ollama":{"OllamaGeneratorChat":{"timeout": 400}}}}'
```

**Actual fix, patch the installed package's hardcoded default directly:**
```bash
python -c "import garak.generators.ollama as m; print(m.__file__)"     # find the file
sed -i 's/"timeout": 30,/"timeout": 400,/' $(python -c "import garak.generators.ollama as m; print(m.__file__)")
grep timeout $(python -c "import garak.generators.ollama as m; print(m.__file__)")   # confirm it took
```

**2. `run.soft_probe_prompt_cap` (meant to limit prompt count per probe) doesn't reliably apply via `--config` either.**
Don't rely on it to shrink a run  budget for full, uncapped probe sizes instead (e.g. 127 prompts for `Ablation_Dan_11_0`, taking several hours on CPU-only hardware).

**3. `--target_type ollama` defaults to `OllamaGeneratorChat`, not `OllamaGenerator`.**
If you ever do get a config override working, target the right class name.

**4. New WSL terminal windows don't remember the venv.**
Always `source ~/garak-env/bin/activate` first.

**5. A pre-existing WSL distro can silently be the default and shadow a new install.**
```bash
wsl -l -v            # list distros, check which is default (run in PowerShell)
wsl -d Ubuntu         # launch a specific one explicitly
```

---

## Config file template (speed/reproducibility settings)

```bash
cat > speedup.yaml << 'EOF'
run:
  soft_probe_prompt_cap: 10
EOF
```
Use with `--config speedup.yaml`. (Remember: the cap itself may not actually apply see Gotcha #2.)

---

## Reports

```bash
# See what got generated
ls ~/.local/share/garak/garak_runs/

# Copy HTML summary report to Windows Desktop to open in a browser
cp ~/.local/share/garak/garak_runs/<prefix>.report.html /mnt/c/Users/aleek/Desktop/

# Extract per-prompt detail (prompt, response, detector score) from the raw .jsonl
sudo apt install jq -y
jq -r 'select(.entry_type=="attempt") | [(.detector_results // {} | to_entries[0].value[0]), (.prompt.turns[0].content.text // .prompt.text // .prompt), (.outputs[0].text // .outputs[0])] | @tsv' ~/.local/share/garak/garak_runs/<prefix>.report.jsonl > /mnt/c/Users/aleek/Desktop/<prefix>_details.tsv
```

### Reading the HTML report
- **DEFCON scale (DC-1 to DC-5):** DC-1 = most severe, DC-5 = safest.
- **Probe score %:** the model's resistance rate lower % = more vulnerable.
- **Tags** (`owasp:llm01`, `avid-effect:security:S0403`, etc.): ready-made mapping to OWASP/AVID frameworks.
- HTML report is summary-only actual prompts/responses only live in the `.jsonl`.

---

## Full example: one complete run, start to finish

```bash
wsl
source ~/garak-env/bin/activate
curl http://localhost:11434/api/tags

cat > speedup.yaml << 'EOF'
run:
  soft_probe_prompt_cap: 10
EOF

python -m garak --config speedup.yaml --target_type ollama --target_name qwen3 \
  --probes dan.Ablation_Dan_11_0 --generations 1 \
  --report_prefix qwen3_dan_ablation --seed 42 --verbose

# in a second terminal, watch it live:
tail -f ~/.local/share/garak/garak.log

# once done:
ls ~/.local/share/garak/garak_runs/
cp ~/.local/share/garak/garak_runs/qwen3_dan_ablation.report.html /mnt/c/Users/aleek/Desktop/
```
