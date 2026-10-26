# darkmux-roles

The role definitions [darkmux](https://github.com/kstrat2001/darkmux) staffs its crew with (coder, reviewer, analyst and the rest), in a repo of their own so an eval can pin the exact role a run used to a commit, instead of copying it.

Moved from darkmux's `templates/builtin/` at commit `9b83ee8e4640778131cdffa4a2825def21d066a3` (darkmux#3363), byte for byte. `web-probe` comes from the biz-scrape-fixture eval, which carried its own copy (`roles/web-probe.*` at `701cf6a1860601ad84c624602f888e3fb3743d39`, the hashes it pins in `evals/easy.json`).

## What is here

| Path | What it is |
|---|---|
| `roles/<id>.json` | The role's manifest: id, family, description, skills, tool palette, escalation contract. |
| `roles/<id>.md` | The role's prompt. |
| `skills/<id>.json` | The skills roles name, with the keywords darkmux routes by. |
| `AUTONOMOUS_DISPATCH_PREAMBLE.md` | Prepended to every specialist role's prompt when darkmux dispatches it. Utility roles (`radio-router`) run without it. |

Roles: analyst, code-reviewer, coder, crawler, design-reviewer, pr-reviewer, radio-host, radio-router, reviewer, test-designer, tool-bench, and web-probe (a tool-less role that answers fixed questions about a website as JSON, with its `output_schema` in the manifest).

## Pinning a role

An eval's workload names its role and this repo at a full commit:

```json
"role": "coder",
"roles": "kstrat2001/darkmux-roles@<40-character commit sha>"
```

The exact field names are settled in darkmux#3363. What the model sees is the preamble followed by the role's prompt, so a pin covers both. A darkmux run records the role it used and the sha256 of the files and of the prompt it assembled. evals.darkmux.com checks those against this repo at the pinned commit (darklyenergized#353).

## Changing a role

Any change is a new commit, and a new commit is a new pin: an eval that wants the change pins the new commit, which makes a new version of that eval. A commit is never rewritten, because pins point at it. Tags (`v1`, `v2`) may name commits for people to read, but a pin always names the commit itself.

## License

MIT, as darkmux. See `LICENSE`.
