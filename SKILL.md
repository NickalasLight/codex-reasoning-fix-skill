---
name: codex-reasoning-fix-skill
description: Diagnose and remediate the local Codex gpt-5.5 shallow/quantized reasoning issue on Windows, macOS, or Linux. Use when codex exec returns wrong answers with suspicious reasoning_output_tokens such as 516, 1034, or 1552, or when applying and verifying the model_instructions_file workaround that removes the final Intermediary updates section from the effective base instructions.
---

# Codex Reasoning Fix

## Problem

Some local Codex `gpt-5.5` installs show shallow or quantized reasoning on hard prompts. The practical signature is a wrong final answer with `turn.completed.usage.reasoning_output_tokens` equal to `516`, `1034`, or `1552`.

The observed workaround is to remove the final `## Intermediary updates` block from the effective `gpt-5.5` base instructions and point Codex at that edited file with a top-level `model_instructions_file` setting in `config.toml`.

Do not rely on editing `models_cache.json` alone. `codex exec` can refresh that cache. The durable part is the top-level `model_instructions_file` override.

## Test Prompt

Use the same prompt for every test run:

```text
Do not use external tools. A black bag contains candies with counts: round apple 7, round peach 9, round watermelon 8; star apple 7, star peach 6, star watermelon 4. Shape is distinguishable by touch before drawing; flavor is not. What is the minimum number of candies to draw to guarantee having apple and peach candies of different shapes, i.e. round apple + star peach or round peach + star apple? Give reasoning and final number. The local project dir is irrelevant for this task, do not consult it.
```

Correct final answer: standalone `21`.

Affected: at least one run has a wrong final answer and `reasoning_output_tokens` is `516`, `1034`, or `1552`.

Fine/fixed: all five runs give final answer `21`, and no wrong-answer run has one of the suspicious token counts.

## Five-Shot Test

### Windows PowerShell

```powershell
$prompt = @'
Do not use external tools. A black bag contains candies with counts: round apple 7, round peach 9, round watermelon 8; star apple 7, star peach 6, star watermelon 4. Shape is distinguishable by touch before drawing; flavor is not. What is the minimum number of candies to draw to guarantee having apple and peach candies of different shapes, i.e. round apple + star peach or round peach + star apple? Give reasoning and final number. The local project dir is irrelevant for this task, do not consult it.
'@

$root = Join-Path $env:TEMP ("codex-reasoning-eval-" + (Get-Date -Format "yyyyMMdd-HHmmss"))
New-Item -ItemType Directory -Path $root | Out-Null

1..5 | ForEach-Object {
  $out = Join-Path $root "run$_.jsonl"
  $err = Join-Path $root "run$_.stderr.txt"
  $prompt | codex exec --json --skip-git-repo-check --ephemeral -s read-only --disable memories -m gpt-5.5 -c model_reasoning_effort=high 1> $out 2> $err
  if ($LASTEXITCODE -ne 0) { throw "run $_ failed: $(Get-Content -Raw $err)" }
}

foreach ($file in Get-ChildItem $root -Filter "run*.jsonl" | Sort-Object Name) {
  $events = Get-Content $file | Where-Object { $_.Trim() } | ForEach-Object { $_ | ConvertFrom-Json }
  $message = @($events | Where-Object type -eq "item.completed" | ForEach-Object item | Where-Object type -eq "agent_message" | Select-Object -Last 1).text
  $usage = @($events | Where-Object type -eq "turn.completed" | Select-Object -Last 1).usage
  [pscustomobject]@{
    Run = $file.BaseName
    ReasoningOutputTokens = $usage.reasoning_output_tokens
    FinalAgentMessage = $message
  }
}

"JSONL directory: $root"
```

### macOS/Linux Shell

```bash
prompt='Do not use external tools. A black bag contains candies with counts: round apple 7, round peach 9, round watermelon 8; star apple 7, star peach 6, star watermelon 4. Shape is distinguishable by touch before drawing; flavor is not. What is the minimum number of candies to draw to guarantee having apple and peach candies of different shapes, i.e. round apple + star peach or round peach + star apple? Give reasoning and final number. The local project dir is irrelevant for this task, do not consult it.'

root="$(mktemp -d "${TMPDIR:-/tmp}/codex-reasoning-eval-XXXXXXXX")"
for i in 1 2 3 4 5; do
  out="$root/run$i.jsonl"
  err="$root/run$i.stderr.txt"
  printf '%s\n' "$prompt" | codex exec --json --skip-git-repo-check --ephemeral -s read-only --disable memories -m gpt-5.5 -c model_reasoning_effort=high >"$out" 2>"$err" || {
    cat "$err" >&2
    exit 1
  }
done

python3 - "$root" <<'PY'
import json, pathlib, sys
root = pathlib.Path(sys.argv[1])
for path in sorted(root.glob("run*.jsonl")):
    events = [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines() if line.strip()]
    messages = [e["item"]["text"] for e in events if e.get("type") == "item.completed" and e.get("item", {}).get("type") == "agent_message"]
    completed = [e for e in events if e.get("type") == "turn.completed"][-1]
    print({"run": path.stem, "reasoning_output_tokens": completed["usage"].get("reasoning_output_tokens"), "final_agent_message": messages[-1] if messages else ""})
print(f"JSONL directory: {root}")
PY
```

## Durable Fix

Use the local Codex home:

- Windows default: `C:\Users\<you>\.codex`
- macOS/Linux default: `~/.codex`
- If `CODEX_HOME` is set, use that instead.

The fix backs up `models_cache.json` and `config.toml`, writes `model-instructions.md` without the final `## Intermediary updates` block, and inserts `model_instructions_file` at top level in `config.toml`.

### Windows PowerShell

```powershell
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
$cache = Join-Path $codexHome "models_cache.json"
$config = Join-Path $codexHome "config.toml"
$instructions = Join-Path $codexHome "model-instructions.md"
$stamp = Get-Date -Format "yyyyMMdd-HHmmss"

Copy-Item $cache "$cache.bak-$stamp" -Force
Copy-Item $config "$config.bak-$stamp" -Force

$models = Get-Content $cache -Raw | ConvertFrom-Json
$model = @($models.models | Where-Object slug -eq "gpt-5.5" | Select-Object -First 1)
if (-not $model) { throw "gpt-5.5 not found in $cache" }

$text = [string]$model.base_instructions
$idx = $text.LastIndexOf("`n## Intermediary updates")
if ($idx -lt 0) { $idx = $text.LastIndexOf("## Intermediary updates") }
if ($idx -lt 0) { throw "Intermediary updates section not found in gpt-5.5 base instructions" }

$clean = $text.Substring(0, $idx).TrimEnd() + "`n"
[System.IO.File]::WriteAllText($instructions, $clean, [System.Text.UTF8Encoding]::new($false))

$tomlPath = $instructions.Replace('\', '\\')
$setting = "model_instructions_file = `"$tomlPath`""
$lines = @(Get-Content $config | Where-Object { $_ -notmatch '^\s*model_instructions_file\s*=' })
$firstTable = -1
for ($i = 0; $i -lt $lines.Count; $i++) {
  if ($lines[$i] -match '^\s*\[') { $firstTable = $i; break }
}
if ($firstTable -lt 0) {
  $newLines = @($setting) + $lines
} elseif ($firstTable -eq 0) {
  $newLines = @($setting, "") + $lines
} else {
  $newLines = @($lines[0..($firstTable - 1)]) + $setting + "" + @($lines[$firstTable..($lines.Count - 1)])
}
[System.IO.File]::WriteAllLines($config, $newLines, [System.Text.UTF8Encoding]::new($false))
```

### macOS/Linux Shell

```bash
python3 <<'PY'
import json, os, pathlib, re, shutil, time

codex_home = pathlib.Path(os.environ.get("CODEX_HOME") or pathlib.Path.home() / ".codex")
cache = codex_home / "models_cache.json"
config = codex_home / "config.toml"
instructions = codex_home / "model-instructions.md"
stamp = time.strftime("%Y%m%d-%H%M%S")

shutil.copy2(cache, cache.with_name(cache.name + f".bak-{stamp}"))
shutil.copy2(config, config.with_name(config.name + f".bak-{stamp}"))

data = json.loads(cache.read_text(encoding="utf-8"))
model = next((m for m in data.get("models", []) if m.get("slug") == "gpt-5.5"), None)
if not model:
    raise SystemExit(f"gpt-5.5 not found in {cache}")

text = model.get("base_instructions") or ""
idx = text.rfind("\n## Intermediary updates")
if idx < 0:
    idx = text.rfind("## Intermediary updates")
if idx < 0:
    raise SystemExit("Intermediary updates section not found in gpt-5.5 base instructions")

instructions.write_text(text[:idx].rstrip() + "\n", encoding="utf-8")

setting = "model_instructions_file = " + json.dumps(str(instructions))
lines = [line for line in config.read_text(encoding="utf-8").splitlines() if not re.match(r"\s*model_instructions_file\s*=", line)]
first_table = next((i for i, line in enumerate(lines) if re.match(r"\s*\[", line)), None)
if first_table is None:
    new_lines = [setting] + lines
elif first_table == 0:
    new_lines = [setting, ""] + lines
else:
    new_lines = lines[:first_table] + [setting, ""] + lines[first_table:]
config.write_text("\n".join(new_lines) + "\n", encoding="utf-8")

print(f"Wrote {instructions}")
print(f"Updated {config}")
PY
```

## Verify The Effective Prompt

Run one non-ephemeral smoke and inspect the persisted session metadata. Expected result: `False`.

### Windows PowerShell

```powershell
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
$before = Get-Date
"Reply exactly OK." | codex exec --json --skip-git-repo-check -s read-only --disable memories -m gpt-5.5 -c model_reasoning_effort=high
$session = Get-ChildItem (Join-Path $codexHome "sessions") -Recurse -File -Filter "*.jsonl" |
  Where-Object LastWriteTime -ge $before |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1
$meta = Get-Content $session.FullName -TotalCount 1 | ConvertFrom-Json
([string]$meta.payload.base_instructions.text).Contains("## Intermediary updates")
```

### macOS/Linux Shell

```bash
before="$(date +%s)"
printf 'Reply exactly OK.\n' | codex exec --json --skip-git-repo-check -s read-only --disable memories -m gpt-5.5 -c model_reasoning_effort=high

python3 - "$before" <<'PY'
import json, os, pathlib, sys
before = int(sys.argv[1])
codex_home = pathlib.Path(os.environ.get("CODEX_HOME") or pathlib.Path.home() / ".codex")
sessions = sorted((p for p in (codex_home / "sessions").rglob("*.jsonl") if p.stat().st_mtime >= before), key=lambda p: p.stat().st_mtime, reverse=True)
if not sessions:
    raise SystemExit("No persisted session found")
meta = json.loads(sessions[0].read_text(encoding="utf-8").splitlines()[0])
base = ((meta.get("payload") or {}).get("base_instructions") or {}).get("text") or ""
print("## Intermediary updates" in base)
print(sessions[0])
PY
```

Finally rerun the five-shot test. Keep the fix only if all five final answers are `21` and no wrong-answer run returns `516`, `1034`, or `1552`.
