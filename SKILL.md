---
name: codex-reasoning-fix-skill
description: Diagnose and remediate the Codex gpt-5.5 shallow/quantized reasoning issue where local codex exec runs return wrong answers with suspicious reasoning_output_tokens such as 516, 1034, or 1552. Use when testing whether a local Codex install is affected, verifying the fix, or applying the model_instructions_file workaround that removes the final Intermediary updates section from the effective base instructions.
---

# Codex Reasoning Fix

## Problem

Some local Codex `gpt-5.5` installs show shallow/quantized reasoning on hard prompts. A strong signal is a wrong answer plus `turn.completed.usage.reasoning_output_tokens` equal to `516`, `1034`, or `1552`.

The observed workaround is to remove the final `## Intermediary updates` block from the effective model instructions and point Codex at that edited file with a top-level `model_instructions_file` setting. Do not rely on editing `models_cache.json` alone; `codex exec` can refresh that cache.

## Five-Shot Test

Run this from PowerShell:

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

Interpretation:

- Affected: any run has a wrong final answer and `reasoning_output_tokens` is `516`, `1034`, or `1552`.
- Fixed/fine: all runs give standalone final answer `21`, and none of the wrong-answer suspicious token counts appear.

## Durable Fix

Use the local Codex home. On Windows this is usually `C:\Users\<you>\.codex`; otherwise use `$env:CODEX_HOME` if set.

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

Ensure `model_instructions_file` is top-level in `config.toml`, before any `[section]` header:

```toml
model_instructions_file = "C:\\Users\\YOUR_USER\\.codex\\model-instructions.md"
```

If a previous attempt put `model_instructions_file` under `[windows]` or another table, move it to the top level. A section-scoped value is not the intended override.

## Verify

Run a non-ephemeral smoke once and inspect the persisted session metadata:

```powershell
$before = Get-Date
"Reply exactly OK." | codex exec --json --skip-git-repo-check -s read-only --disable memories -m gpt-5.5 -c model_reasoning_effort=high
$session = Get-ChildItem (Join-Path $codexHome "sessions") -Recurse -File -Filter "*.jsonl" |
  Where-Object LastWriteTime -ge $before |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1
$meta = Get-Content $session.FullName -TotalCount 1 | ConvertFrom-Json
([string]$meta.payload.base_instructions.text).Contains("## Intermediary updates")
```

Expected result: `False`.

Finally rerun the five-shot test. Keep the fix only if the final answers are `21` and suspicious wrong-answer token counts disappear.
