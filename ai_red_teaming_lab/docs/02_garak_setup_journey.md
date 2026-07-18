# Setting Up Garak for Local LLM Red Teaming: A Field Guide

Context: CPU only Windows 11 laptop (i7-1255U, 16GB RAM, Intel Iris Xe integrated graphics, no dedicated GPU), with Ollama already installed and several local models available (qwen3, gemma4:e4b, huihui_ai/qwen3.5-abliterated:9b, among others). Goal: get NVIDIA's Garak LLM vulnerability scanner running against those local models.

This document is written the way it actually happened, mistakes included, because the mistakes are the useful part. If you are following in these footsteps, this will save you from repeating them.

## TL;DR: the path that actually works

Skip straight to Part 2 (WSL + Ubuntu). Part 1 documents the native Windows attempt and why it is not worth the effort. Read it only if you are curious why WSL is the recommended route.

## Part 1: The native Windows attempt (abandoned, read for context, not to follow)

### Installing Python

Python was not installed. Installed via winget:

```
winget install Python.Python.3.12
```

Problem: after installing, `python --version` returned "Python was not found, run without arguments to install from the Microsoft Store". Windows' built in Store alias stub was shadowing the real install.

Fix:

1. Close and reopen PowerShell completely (a fresh process is needed to pick up the updated PATH).
2. If that does not clear it, go to Settings, Apps, Advanced app settings, App execution aliases, and turn off the toggles for `python.exe` and `python3.exe`.

### Installing Garak: hit a Rust build wall

```
python -m pip install garak
```

Problem 1: failed while building `litellm` (a Garak dependency) from source:

```
error: Failed to build a native library through cargo
```

Cargo, the Rust package manager, was not installed or not on PATH. No prebuilt wheel existed for this package, platform, and Python combination, so pip fell back to compiling it from source, which needs a Rust toolchain.

Attempted fix:

```
winget install Rustlang.Rustup
# close and reopen PowerShell
rustup default stable
cargo --version   # confirm it worked
python -m pip install garak   # retry
```

Problem 2: retrying the install got further, then failed again compiling Rust crates (`quote`, `proc-macro2`, `target-lexicon`) with linker errors. Rust's default Windows toolchain needs the MSVC linker (`link.exe`), which only comes from Visual Studio Build Tools, not from Rust itself.

The fix would have been:

```
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --override "--wait --passive --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

This is a 4 to 6GB download. At this point it was decided this was not worth it. This whole chain of problems, Rust needed, then MSVC needed, then a multi gigabyte toolchain, is a Windows specific headache that mostly does not exist on Linux, where these packages ship prebuilt binaries. Decision: switch to WSL (Windows Subsystem for Linux) instead of fighting the native Windows build chain.

### Cleanup before switching (reclaim disk space)

```
rustup self uninstall
python -m pip cache purge
Get-ChildItem $env:LOCALAPPDATA\Temp -Filter "pip-*" -Directory | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path $env:TEMP\* -Recurse -Force -ErrorAction SilentlyContinue
Get-PSDrive C   # check space reclaimed
```

The Visual Studio Build Tools were deliberately never installed, avoiding that multi gigabyte cost entirely.

## Part 2: The WSL + Ubuntu path (this is the one to follow)

### Step 1: Install WSL with Ubuntu

In PowerShell, as Administrator:

```
wsl --install -d Ubuntu
```

This installs WSL2 and Ubuntu, then prompts a restart. Restart the PC.

### Step 2: First Ubuntu launch

After restart, Ubuntu opens automatically and asks you to create a Linux username and password, independent of your Windows login.

### Step 3: Enable mirrored networking

This lets WSL and Windows share localhost, so WSL can reach Ollama running on Windows without extra networking work.

In PowerShell:

```
notepad $env:USERPROFILE\.wslconfig
```

Notepad offers to create the file if it does not exist. Paste in:

```
[wsl2]
networkingMode=mirrored
```

Save and close.

### Step 4: Make Ollama listen on all interfaces

By default Ollama only listens on Windows' own loopback. In PowerShell:

```
[System.Environment]::SetEnvironmentVariable('OLLAMA_HOST','0.0.0.0',[System.EnvironmentVariableTarget]::User)
```

Then fully quit Ollama (right click its system tray icon, then Quit) and reopen it so it picks up the new variable.

### Step 5: Restart WSL to apply networking changes

```
wsl --shutdown
```

Reopen Ubuntu from the Start menu afterward.


### Step 6: Set up Python and Garak inside Ubuntu

```
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv -y
python3 -m venv ~/garak-env
source ~/garak-env/bin/activate
pip install garak
```

This time, no Rust or MSVC issues at all. Linux wheels exist for everything Garak needs.

One thing that will bite you repeatedly: every time you open a new WSL terminal window, you must reactivate the venv before garak will work:

```
source ~/garak-env/bin/activate
```

You will know it worked because the prompt shows `(garak-env)` in front.

### Step 7: Confirm WSL can actually reach Ollama

```
curl http://localhost:11434/api/tags
```

This should return JSON listing your installed models. If it hangs or errors, the mirrored networking and `OLLAMA_HOST` setup from Steps 3 and 4 did not take. Recheck those before going further.

## Part 3: Running Garak, and the "is it stuck" saga

### The first run

```
python -m garak --model_type ollama --model_name qwen3 --probes dan,promptinject --generations 2 --report_prefix qwen3_baseline
```

Note: `--model_type` and `--model_name` are deprecated aliases for `--target_type` and `--target_name` in current Garak. Both work, but the newer flags are the ones to reach for going forward.

It looked stuck. The progress bar sat at 0/127, 0/2 for a long time with no visible movement. Here is the actual troubleshooting path, because most of these dead ends are worth knowing about.

1. Checked `top` inside WSL: showed near zero CPU usage. This was a red herring. The actual model inference happens inside the Windows hosted Ollama app, not inside WSL. WSL just sends an HTTP request and waits, so it will always look idle there regardless of what is really happening.
2. Checked Windows Task Manager for `ollama.exe`: initially only saw lightweight helper processes (`ollama.exe`, `ollama app.exe`) at 0 percent CPU and 10 to 30MB memory. The actual worker process is a separate one, `llama-server.exe`, which only appears while a generation is actively running and uses much more CPU and RAM, in this case about 49 percent CPU and 4.7GB RAM once caught live. Check Task Manager for both process names, since the actual worker is easy to miss on a quick glance.
3. Tested raw connectivity directly, bypassing the running scan:

   ```
   time curl http://localhost:11434/api/generate -d '{"model": "qwen3", "prompt": "Say hello in one sentence.", "stream": false}'
   ```

   The first attempt appeared to hang with zero output, most likely queued behind the scan's own in flight request (Ollama generally handles one request at a time per model). A later attempt succeeded, taking 35.5 seconds for a trivial one line response. That number turned out to be the key clue for the real bug, covered in Part 4.
4. Miscalculated elapsed time at one point due to a timestamp mix up. Ollama's `created_at` field is UTC, while Garak's own log uses local time. Always check what timezone you are comparing before concluding something has been running for hours.

### Reality check on speed

CPU only inference on an integrated graphics laptop is genuinely slow, and thinking capable models (qwen3 here) add a chain of thought reasoning pass before the actual answer, inflating response time further. A single trivial prompt took 35+ seconds. Longer, more complex jailbreak style prompts can take 1 to 3+ minutes each. Budget accordingly, a full multi probe scan can realistically take several hours on hardware like this.

## Part 4: The real bug, Garak's default timeout is shorter than this hardware's generation speed

Once `--verbose` was added and the log tailed in a second terminal (`tail -f ~/.local/share/garak/garak.log`), the actual problem became visible:

```
receive_response_headers.failed exception=ReadTimeout(TimeoutError('timed out'))
...
INFO Backing off _call_model(...) for 30.4s (garak.exception.GeneratorBackoffTrigger)
```

This pattern repeated forever. Garak's Ollama generator has a hardcoded 30 second timeout, and this hardware needed 35+ seconds per response, so every single request timed out right before Ollama would have answered, triggered a backoff and retry, and repeated indefinitely. Not stuck, caught in an infinite retry loop caused by a timeout that was simply too short for this hardware.

### Attempted fix 1: YAML config override (did not work)

```
cat > speedup.yaml << 'EOF'
run:
  soft_probe_prompt_cap: 10
generators:
  ollama:
    OllamaGenerator:
      timeout: 300
EOF

python -m garak --config speedup.yaml --target_type ollama --target_name qwen3 --probes dan.Ablation_Dan_11_0 --generations 1 --report_prefix qwen3_dan_ablation --seed 42
```

The log still showed `timeout=30`. No effect.

### Attempted fix 2: --generator_options CLI flag, matching Garak's own documented example (also did not work)

```
python -m garak --config speedup.yaml --generator_options '{"generators":{"ollama":{"OllamaGeneratorChat":{"timeout": 400}}}}' --target_type ollama --target_name qwen3 --probes dan.Ablation_Dan_11_0 --generations 1 --report_prefix qwen3_dan_ablation --seed 42 --verbose
```

Still `timeout=30` in the logs. Two different documented configuration methods, same result. This points to a real bug in this Garak version's config loading path for the Ollama generator. There is a corresponding open GitHub issue, #1166, about generator config values not being picked up correctly.

### What the actual source code showed

Fetched `garak/generators/ollama.py` directly from GitHub and found the relevant bits:

```python
class OllamaGenerator(Generator):
    DEFAULT_PARAMS = Generator.DEFAULT_PARAMS | {
        "timeout": 30,
        "host": "127.0.0.1:11434",
    }
    ...
    DEFAULT_CLASS = "OllamaGeneratorChat"
```

Two things this explained. First, `--target_type ollama` defaults to the Chat variant of the generator, which is why the first config attempt, targeting the plain `OllamaGenerator` key, was doubly wrong, but even correcting the class name in fix 2 still did not help, confirming the config path itself is broken, not just the key name. Second, the hardcoded default really is 30 seconds, baked directly into the installed package.

### The fix that actually worked: patch the installed package directly

Since neither documented config method took effect, the hardcoded default was edited directly in the venv's own copy of the file. This is fully safe, it is just the local install, easily reversible, and does not touch anything outside this project.

```
# Find the exact file path
python -c "import garak.generators.ollama as m; print(m.__file__)"

# Patch the default timeout from 30 to 400 seconds
sed -i 's/"timeout": 30,/"timeout": 400,/' $(python -c "import garak.generators.ollama as m; print(m.__file__)")

# Confirm it took
grep timeout $(python -c "import garak.generators.ollama as m; print(m.__file__)")
```

Rerunning the exact same scan command, this time the log showed `timeout=400`, the request actually completed (`HTTP/1.1 200 OK` after about 75 seconds), and the scan moved on to the next prompt instead of looping forever.

Lesson: when a tool's documented configuration mechanism does not work, and that has been verified with two different correct approaches, checking the actual source code for hardcoded defaults, and patching them directly in your own venv if needed, is a legitimate and fast way forward. Not elegant, but effective, and fully contained to the local environment.

## Part 5: Limiting scan size, soft_probe_prompt_cap (works in theory, did not work in practice)

Garak has a setting meant to cap how many prompts a large probe uses, useful for cutting down runtime:

```
run:
  soft_probe_prompt_cap: 10
```

used via `--config speedup.yaml`. In this case, this also silently failed to apply. A run expected to cap at 10 prompts kept going and reached 63/127, 50 percent, partway through, confirming the full 127 prompt set was running regardless of the cap setting in the config file.

Since it was already progressing correctly at that point, just uncapped, the full run was left to complete rather than chasing down another config loading bug. Practical takeaway: on this Garak version, do not rely on `--config` for either the generator timeout or the prompt cap. Budget for full size, uncapped runs, hours, not minutes, and treat them as background or overnight jobs.

### Useful commands for managing long runs

Watch live activity in a second terminal:

```
tail -f ~/.local/share/garak/garak.log
```

Add verbosity to the main run:

```
--verbose
```

Fix the random sample seed for reproducibility. This does not solve the cap issue, but it ensures that if a cap does apply, the same subset comes back every time, which is useful for repeatable, auditable results:

```
--seed 42
```

Note: different seed values are not a substitute for true chunking. They give independent random draws that can overlap and do not guarantee full, non overlapping coverage of a probe's prompt list. If genuinely non overlapping chunks of a large probe are needed, that requires a custom script slicing the probe's actual prompt list, not something achievable through seeds alone.

## Part 6: Reading the results

### Where reports land

Garak (0.14+) auto generates both a raw `.jsonl` log and a formatted `.html` summary report per run:

```
ls ~/.local/share/garak/garak_runs/
```

### Viewing the HTML report

Copy it to somewhere Windows can open directly:

```
cp ~/.local/share/garak/garak_runs/qwen3_dan_ablation.report.html /mnt/c/Users/<yourusername>/Desktop/
```

Then double click it on the Desktop to open in your browser.

### Interpreting the report

DEFCON style rating (DC-1 to DC-5): borrowed from the military alert scale. DC-1 is most severe, DC-5 is safest. A "DC-2 / Very High Risk" result means the model performed poorly against that probe.

Probe score percentage: roughly, the model's resistance rate, the percentage of attempts where it successfully held its safety behavior. Lower means more vulnerable.

Tags, for example `owasp:llm01`, `avid-effect:security:S0403`, `payload:jailbreak`: map the specific finding to established frameworks, OWASP's LLM Top 10, the AVID vulnerability taxonomy. This is the ready made bridge between a raw technical result and governance or audit language.

### Getting the actual prompt by prompt detail (the HTML report is summary only)

The HTML report does not contain the individual prompts and model responses, only the raw `.jsonl` does. Extract a readable version with `jq`:

```
sudo apt install jq -y

jq -r 'select(.entry_type=="attempt") | [(.detector_results // {} | to_entries[0].value[0]), (.prompt.turns[0].content.text // .prompt), .outputs[0]] | @tsv' ~/.local/share/garak/garak_runs/qwen3_dan_ablation.report.jsonl > /mnt/c/Users/<yourusername>/Desktop/qwen3_dan_details.tsv
```

This produces a spreadsheet ready file with the detector's score for that attempt, closer to 1.0 generally means the jailbreak succeeded, closer to 0.0 means the model resisted, worth spot checking a few rows manually rather than trusting it blindly, the actual prompt text, and the model's literal response. Sort by score descending in Excel to find the specific prompts that actually broke the model. This is the real evidence for a writeup.

## Quick reference: commands used day to day

Start of every session:

```
# Open Ubuntu (Start menu, or from PowerShell:)
wsl

# Confirm you're in Ubuntu, not another distro
cat /etc/os-release

# Activate the venv (needed in every new terminal window)
source ~/garak-env/bin/activate

# Confirm WSL can reach Ollama
curl http://localhost:11434/api/tags
```

Run a probe:

```
python -m garak --target_type ollama --target_name qwen3 --probes dan.Ablation_Dan_11_0 --generations 1 --report_prefix qwen3_dan_ablation --seed 42 --verbose
```

Watch it live (second terminal):

```
tail -f ~/.local/share/garak/garak.log
```

After it finishes:

```
ls ~/.local/share/garak/garak_runs/
cp ~/.local/share/garak/garak_runs/<prefix>.report.html /mnt/c/Users/<yourusername>/Desktop/
```

## Summary of bugs and gotchas worth remembering

1. Native Windows Garak install drags in a Rust compiler and then a full Visual Studio C++ Build Tools install. Not worth it, go straight to WSL.
2. A preexisting WSL distro, for example an old Kali install, can silently become the default and shadow a freshly installed one. Check `wsl -l -v` and launch explicitly with `wsl -d <name>` if unsure.
3. The venv needs reactivating (`source ~/garak-env/bin/activate`) in every new terminal window.
4. Ollama's actual inference process is `llama-server.exe` on the Windows side. Check Task Manager for that name, not just `ollama.exe`, to confirm real activity.
5. Garak's Ollama generator has a hardcoded 30 second timeout that is too short for slow, CPU only hardware, and in this version, neither the documented `--config` YAML method nor the `--generator_options` CLI method actually overrides it. The reliable fix is patching the `DEFAULT_PARAMS` timeout directly in the installed package's source file.
6. `run.soft_probe_prompt_cap`, meant to limit prompt count per probe, also did not reliably apply via `--config` in this version. Budget for full, uncapped runs instead.
7. The HTML report is a summary dashboard only. Actual prompt and response transcripts live in the raw `.jsonl` file.
