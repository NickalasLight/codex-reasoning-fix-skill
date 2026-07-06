# Codex Reasoning Fix

## Contributing Changes

Public contributions are welcome. If you are not Nickalas Light and want to update this repo:

1. Fork the repository.
2. Create a branch in your fork with a focused name, such as `fix/eval-parser` or `docs/windows-notes`.
3. Make a small, reviewable change. Prefer one fix or documentation improvement per pull request.
4. Do not include secrets, API keys, private logs, customer data, cookies, or machine-specific private paths.
5. If your change is based on analysis, include the exact command, Codex version/model, platform, JSONL/event fields, and any relevant sanitized output needed to reproduce it.
6. Run the relevant local verification from `SKILL.md` when possible. If you cannot run it, say why in the PR body.
7. Open a pull request against `NickalasLight/codex-reasoning-fix-skill:main` and describe:
   - what changed
   - why it changed
   - how you verified it
   - any remaining uncertainty

Maintainers will review before merging. Public contributors do not need direct write access to propose changes.

## About

This repository contains a Codex skill for diagnosing and remediating the local Codex `gpt-5.5` shallow or quantized reasoning issue, especially runs with suspicious `reasoning_output_tokens` values such as `516`, `1034`, or `1552`.

The full skill instructions and verification commands are in [SKILL.md](./SKILL.md).
