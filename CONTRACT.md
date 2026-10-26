# The contract between darkmux and darkmux-roles

**Draft for review (darkmux#3363, darklyenergized#353, #356). Nothing here is built yet.**

What darkmux may assume about this repo, what it must do with it, and what a run records, so that evals.darkmux.com can check a run used the official inputs. The rule (Kain, 2026-10-26): every input that shapes a run is hashed by darkmux and verified against a pinned public source. Identity is the content hash, not the location: a file byte-identical in two places is the same input.

## What pins a run

A run is official when every input that shapes it is pinned or is a declared variable, and the owner has approved it or it has been reproduced.

| Layer | What it covers | How it is pinned |
|---|---|---|
| darkmux itself | Detection, grading, tool handling, prompt assembly: anything that changes with the code | The release version. Only a published release counts; a dev build or fork is never official, whatever version it reports. |
| Roles | Role prompt, role config (`.json`), preamble | A commit of this repo (section 3). |
| The eval | Workload, fixture files, expected answers | The fixture's commit, as now. |
| Everything else a user can change without rebuilding | `~/.darkmux/roles/`, the preamble override, `identity.md`, workspace notes, profiles and config, env vars such as `DARKMUX_IDENTITY_PATH` | Not pinned: a lab run ignores it (section 4). |
| Declared variables | The model, its quantization, the hardware, the runtime and its settings | Recorded, not pinned. These are what an eval compares. |

**Why ignore, not hash.** Ignoring the home folder for a lab run makes local drift impossible (the stale `web-probe.md` of 2026-10-24 could not have happened). Hashing local files would only report the drift after the run.

**What hashes cannot do.** Every hash in a bundle is reported by the contributor's own machine. They catch honest mistakes and make a run's claim exact, but a modified darkmux can report an official version and the right hashes. Deliberate spoofing is caught by the site, not the bundle. evals.darkmux.com accepts only known releases, holds contributor uploads for the owner's review under its terms, and marks a run reproduced only when an independent run agrees.

## 1. What the repo holds

| Path | Required | What it is |
|---|---|---|
| `roles/<id>.json` | yes | The role's config: family, tool palette, output schema, skills, escalation contract. `id` equals the file name; `schema_version` is the role schema darkmux reads. |
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
    "coder": { "config": "…sha256…", "md": "…sha256…", "family": "specialist" },
    "web-probe": { "config": "…", "md": "…", "family": "specialist" }
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
5. **Read nothing user-configurable for a lab run** beyond the declared variables (see "What pins a run"). In the prompt that means:
   - no operator identity (`identity.md`, added today to local dispatches);
   - no workspace notes (the no-git note).

   If a run needs a workspace note to work, the eval's fixture is broken, and the run should fail rather than carry a per-machine prompt. Open question 2 covers this.
6. **Hash the prompt** that reaches the model, at the point darkmux already captures it (`capture_assembled_system_prompt`). Record that hash.

## 5. What a run records (the bundle)

darkmux reports; it does not judge. darkmux writes what it knows, and evals.darkmux.com decides what to check and what a run counts as. A field darkmux cannot fill is `null`, never left out or guessed.

**Metadata is separate from measurements** (Kain, 2026-10-26). Hashes and facts about how a run was made are not metrics, so each run gets a `metadata` object beside its measurements. Measurements stay as strict as now. `metadata` is open, so darkmux can add provenance without a site deploy first, within limits that keep anything identifying out:

- a key is a plain name (`[a-z][A-Za-z0-9]{0,59}`) and not a word that names a person or machine (the site's existing list: user, host, email, serial and the rest);
- a value is `null`, true/false, a number, or a token: up to 128 characters, no whitespace, nothing that looks like a file path, such as a sha256, a version, or `owner/repo@sha`;
- objects nest at most two levels, with a cap on the number of keys;
- a key the site knows is checked strictly; one it doesn't know is stored and never shown, and never counts toward a run being official.

`metadata` arrives once, in bundle format 6, which the site reads and deploys before darkmux writes it. After that, new provenance needs no format bump. Existing provenance fields (`darkmuxCommit`, `darkmuxBuild`, `workloadSha256`) stay where they are. Per run:

```json
"metadata": {
  "role": {
    "id": "coder",
    "roles": "kstrat2001/darkmux-roles@<sha>",
    "config": "<sha256 of roles/coder.json>",
    "md": "<sha256 of roles/coder.md>",
    "preamble": "<sha256 of the preamble, or null for a non-specialist role>",
    "prompt": "<sha256 of the assembled system prompt>"
  }
}
```

What the site does with them (its side, not darkmux's): it reads the three files at the pinned commit, checks them against `config`, `md` and `preamble`, rebuilds the prompt by section 4, and checks it against `prompt`. A run whose four match shows as using the pinned role. One that doesn't, or doesn't report them, can't be official. The site also requires `darkmuxBuild: "release"` (already in the bundle since format 3) and a known release version.

## 6. darkmux releases

- Each release compiles in one default pin (`DEFAULT_ROLES = "kstrat2001/darkmux-roles@<sha>"`) and an offline copy of that commit's files. These are used when a workload names no pin, or when there is no network and the cache is empty.
- The offline copy is verified against the same index, so it is the same input as the pinned files.
- A release changes which roles it defaults to only by changing that pin.

## 7. Existing runs

Runs from before the pin (darkmux 5.6.0 and 5.7.0) are verified from what was official then:
- `coder`: from darkmux's `templates/builtin/` at the run's release commit;
- `web-probe`: from the copy the biz-scrape-fixture eval pins in `evals/easy.json`.

Both are byte-identical to this repo's first commits, so evals.darkmux.com can record those hashes and point those runs here.

That proves only which files were official, not what the model saw. darkmux up to 5.7.1 reads `~/.darkmux/roles/<id>.md` before its built-in copy, never reads the fixture's role, and records no prompt hash. On 2026-10-24 a stale `~/.darkmux/roles/web-probe.md` (sha `b0438c…`) silently replaced the pinned one for 19 biz-scrape runs (darkmux#3363). So a backfilled run is marked "official files, prompt not reported", never "verified". Only a run that reports the four hashes in section 5 counts as verified.

## 8. Changing a role

A pull request here makes a new commit. An eval that wants it pins the new commit, which makes a new version of the eval on evals.darkmux.com. A new darkmux release that wants it as its default moves `DEFAULT_ROLES`. Nothing changes under a run that already exists.

## Open questions

1. **Skills:** they're routing only today, so they aren't hashed. If a skill ever contributes prompt text, it joins section 5.
2. **Workspace notes:** fail the run (as proposed above), or allow a fixed, published list of notes and include them in the rebuild?
3. **The full list of user-configurable inputs** a lab run reads today (rules, crew or mission config, profiles, env vars), each to be ignored for lab runs or named as a declared variable. Comes from an audit of darkmux.
4. **Fetching:** `raw.githubusercontent.com` per file, or one tarball per commit?
