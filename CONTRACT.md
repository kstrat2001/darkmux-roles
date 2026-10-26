# The contract between darkmux and darkmux-roles

**Draft for review (darkmux#3363, darklyenergized#353, #356). Nothing here is built yet.**

What darkmux may assume about this repo, what it must do with it, and what a run records, so that evals.darkmux.com can check a run used the official inputs. The rule (Kain, 2026-10-26): every input that shapes a run is hashed by darkmux and verified against a pinned public source. Identity is the content hash, not the location: a file byte-identical in two places is the same input.

## 1. What the repo holds

| Path | Required | What it is |
|---|---|---|
| `roles/<id>.json` | yes | Role manifest. `id` equals the file name; `schema_version` is the role schema darkmux reads. |
| `roles/<id>.md` | yes | Role prompt, UTF-8. |
| `AUTONOMOUS_DISPATCH_PREAMBLE.md` | yes | One preamble for every specialist role. |
| `skills/<id>.json` | when a role names the skill | Routing metadata (keywords). Not part of any prompt. |
| `roles.json` | yes, from contract 1 | The index (section 2). |

`<id>` is `[a-z0-9][a-z0-9-]{0,63}`. Nothing else in the repo is read.

## 2. The index, `roles.json`

```json
{
  "contract": 1,
  "preamble": { "path": "AUTONOMOUS_DISPATCH_PREAMBLE.md", "sha256": "…" },
  "roles": {
    "coder": { "json": "…sha256…", "md": "…sha256…", "family": "specialist" },
    "web-probe": { "json": "…", "md": "…", "family": "specialist" }
  }
}
```

- `contract` is this document's major version. darkmux refuses a repo whose `contract` it doesn't know.
- Every file a role needs is listed with its sha256. A commit whose index doesn't match its own files is invalid: a check in this repo (CI or a script) fails it before it lands.
- `family` is copied from the manifest so a reader knows whether the preamble applies without parsing it.

## 3. How a workload names a role

```json
"role": "coder",
"roles": "kstrat2001/darkmux-roles@<40-character commit sha>"
```

- The pin is a full commit, never a branch or tag. A commit here is never rewritten.
- Without `roles`, darkmux uses the roles pin compiled into its release (section 6). evals.darkmux.com treats such a run as pinned to that release's default, which it can verify.

## 4. What darkmux does with a pin

1. **Fetch** `roles.json` and the role's files at the commit, from `raw.githubusercontent.com` or from a content-addressed cache (`<root>/cache/roles/<sha>/`).
2. **Verify** every file against the index before use. A mismatch, or a file over 64 KB, refuses the run.
3. **Ignore local overrides for a pinned run.** `<root>/roles/` and `<root>/AUTONOMOUS_DISPATCH_PREAMBLE.md` are not read when a pin is given. Missions and ad-hoc dispatches keep using overrides as today.
4. **Assemble the system prompt**, exactly as darkmux does today:
   - for a specialist role: `trim_end(preamble) + "\n\n" + role_md`;
   - for any other role family: `role_md`.
5. **Add nothing per-machine to a pinned eval run's prompt:**
   - no operator identity (`identity.md`, added today to local dispatches);
   - no workspace notes (the no-git note).

   If a run needs a workspace note to work, the eval's fixture is broken, and the run should fail rather than carry a per-machine prompt. Open question 2 covers this.
6. **Hash the prompt** that reaches the model, at the point darkmux already captures it (`capture_assembled_system_prompt`). Record that hash.

## 5. What a run records (the bundle)

Per run, in a new bundle format version:

```json
"role": {
  "id": "coder",
  "roles": "kstrat2001/darkmux-roles@<sha>",
  "json": "<sha256 of roles/coder.json>",
  "md": "<sha256 of roles/coder.md>",
  "preamble": "<sha256 of the preamble, or null for a non-specialist role>",
  "prompt": "<sha256 of the assembled system prompt>"
}
```

evals.darkmux.com reads the three files at the pinned commit, checks them against `json`, `md` and `preamble`, rebuilds the prompt by section 4, and checks it against `prompt`. A run counts as official only if all four match.

## 6. darkmux releases

- Each release compiles in one default pin (`DEFAULT_ROLES = "kstrat2001/darkmux-roles@<sha>"`) and an offline copy of that commit's files. These are used when a workload names no pin, or when there is no network and the cache is empty.
- The offline copy is verified against the same index, so it is the same input as the pinned files.
- A release changes which roles it defaults to only by changing that pin.

## 7. Existing runs

Runs from before the pin (darkmux 5.6.0 and 5.7.0) are verified from what was official then:
- `coder`: from darkmux's `templates/builtin/` at the run's release commit;
- `web-probe`: from the copy the biz-scrape-fixture eval pins in `evals/easy.json`.

Both are byte-identical to this repo's first commits, so evals.darkmux.com can record those runs' hashes and point them here. These are marked "verified from release", since darkmux did not report them.

## 8. Changing a role

A pull request here makes a new commit. An eval that wants it pins the new commit, which makes a new version of the eval on evals.darkmux.com. A new darkmux release that wants it as its default moves `DEFAULT_ROLES`. Nothing changes under a run that already exists.

## Open questions

1. **Skills:** they're routing only today, so they aren't hashed. If a skill ever contributes prompt text, it joins section 5.
2. **Workspace notes:** fail the run (as proposed above), or allow a fixed, published list of notes and include them in the rebuild?
3. **Other files a lab run reads** that shape the model's input or tools: rules, crew or mission config, profiles. darkmux#3363 lists them; each joins this contract or is shown not to apply to eval runs.
4. **Fetching:** `raw.githubusercontent.com` per file, or one tarball per commit?
