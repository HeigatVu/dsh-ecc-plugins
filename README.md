# ecc-dsh — Everything Claude Code on DeepSeek Harness

A DeepSeek Harness (DSH) bundle that mounts an ECC checkout into a profile in one
step. It replaces the two manual steps in
ECC's DeepSeek Harness guide
(<https://github.com/affaan-m/ECC/blob/main/docs/DEEPSEEK-HARNESS-GUIDE.md>): per-skill
symlinks into `~/.dsh/skills`, and a hand-written profile patch entry for the hooks
bridge.

## What it mounts

| Row | Plugin | Effect |
|-----|--------|--------|
| `ecc-skills` | `@deepseek-ai/dsh-skill-filesystem` (ships with DSH) | Serves `<ECC_ROOT>/skills` (311 skills) as a host-plane skill provider, so the catalog is in the merged skill set of every agent preset. No files are copied or linked. |
| `ecc-hooks` | `@deepseek-ai/dsh-hooks-claude-code` (ships with DSH) | Runs the generated ECC command hooks (GateGuard, observe, governance capture, `SessionStart` context, `Stop` checks) through DSH's Claude Code hooks bridge. |

## Prerequisites

- DeepSeek Harness with the `skill` registry and the hooks bridge (0.2.0-rc.2
  verified here; `dsh --version`).
- An ECC checkout kept in place: the hooks config embeds its absolute path.
- Node.js `^22.19.0 || >=24`.

## 1. Generate the hooks config

```bash
node <ECC_ROOT>/scripts/dsh/generate-hooks-config.js \
  --ecc-root <ECC_ROOT> \
  --out "${DSH_HOME:-$HOME/.dsh}/ecc/hooks-dsh.json"
```

The generator prints the events it skipped (`PreCompact`, `PostToolUseFailure`,
`SessionEnd` have no bridge counterpart). Re-run it after `git pull` in the ECC
checkout, and after moving the checkout.

## 2. Install the bundle

From the DSH GUI: **Plugins → install bundle**, target this directory. From an
agent session, the `plugin_manager` tool with `install_bundle` and the same
absolute path. The equivalent CLI step installs the package into the profile:

```bash
cd ~/.dsh
dsh plugin --profile web add <ECC_ROOT>/plugins/dsh           # local checkout
dsh plugin --profile web add github:HeigatVu/dsh-ecc-plugins  # from this repository
```

Either spec puts `ecc-dsh` in the profile's `dependencies`. It must also be listed
in `dsh.profile.bundles` (`~/.dsh/profiles/web/package.json`), because only the
listed bundles are mounted; add the entry if the installer did not. Installing with
`-D` files the package under `devDependencies` and leaves it out of `bundles`,
which mounts no rows at all.

## 3. Override the checkout path (optional)

Both rows resolve the ECC checkout from `ECC_ROOT` and otherwise fall back to the
path baked into `cordis.patch.yml`. Set `ECC_ROOT` before launching DSH if the
checkout moves, or edit the two fallback paths after moving it.

## Verify

```bash
# Compose check (fresh process). Keep stderr visible: a rejected or dropped
# patch entry is reported there while the exit status stays 0.
cd "${DSH_HOME:-$HOME/.dsh}" && dsh --profile web --dump-config > /tmp/ecc-dump.yml
grep -A6 'id: ecc-skills' /tmp/ecc-dump.yml

# Boot proof: a profile whose bundles are base + headless + ecc-dsh boots and
# answers. The bundle test profile also proves the !!js expressions evaluate.
dsh --profile ecc-bundle-test 'Reply with exactly: BOOT-OK'
```

## Verified

2026-10-16, DSH 0.2.0-rc.2, this bundle installed into the `web` profile:

- Compose: `dsh --profile web --dump-config` and
  `dsh --profile ecc-bundle-test --dump-config` both exit 0 with empty stderr and
  carry `ecc-skills` plus `ecc-hooks` with their resolved config.
- Boot: `dsh --profile ecc-bundle-test 'Reply with exactly: BOOT-OK'` prints
  `BOOT-OK` and exits 0 with the bundle as the only non-first-party layer.
- Hooks: the `ecc-hooks` row fires. A marker `SessionStart` command wrote its file,
  and an instrumented boot that ran the real generated hook command (the
  `SessionStart` entry of `hooks-dsh.json`, `run-with-flags.js` chain included)
  reported `REAL_RC=0`.
- Environment seen inside a hook process: `HOME=/home/ducvu`, `PWD=<launch dir>`,
  and nothing else ECC could key on — DSH exports no session id or ECC variable to
  hooks. ECC's session-scoped state therefore stays idle in DSH
  (`[SessionStart] No session ID available; skipping observer lease registration`)
  and exits 0; the hooks that do write state are the ones described under State and
  permissions.

## State and permissions

DSH sandboxes hook processes exactly like tool commands: a hook may only write
inside the process workspace root (`process.cwd()`, the directory DSH was
launched from) plus platform temp areas. Everything else is read-only, `~`
included. ECC's hooks default to state roots outside any workspace — `~/.claude`
(`ECC_AGENT_DATA_HOME`), `~/.gateguard` (`GATEGUARD_STATE_DIR`), and
`~/.local/share/ecc-homunculus` (`XDG_DATA_HOME` or `CLV2_HOMUNCULUS_DIR`) — so a
sandboxed session logs, and then swallows, failures like:

```text
[SessionStart] Error: Failed to create directory '/home/ducvu/.claude/session-data': ENOENT ...
mkdir: cannot create directory '/home/ducvu/.local/share/ecc-homunculus/projects/<id>/instincts': Read-only file system
[Fact-Forcing Gate] GateGuard state could not be persisted; allowing this operation to avoid a permanent retry loop.
```

Hooks fail open, so sessions still run — but GateGuard's first-touch denial never
fires and the continuous-learning, instinct and metrics state is not recorded.
Two ways out:

```bash
# A. Keep ECC state inside the launch root. Works under the default policy when
#    DSH is launched from ~/.dsh; from a project directory use a path inside it,
#    which makes state per project.
export ECC_AGENT_DATA_HOME=${DSH_HOME:-$HOME/.dsh}/ecc/state/agent-data
export CLV2_HOMUNCULUS_DIR=${DSH_HOME:-$HOME/.dsh}/ecc/state/homunculus
export GATEGUARD_STATE_DIR=${DSH_HOME:-$HOME/.dsh}/ecc/state/gateguard
```

B. Run sessions with a wider file policy (`danger-full-access`) so the hooks
write to their usual `~` locations. That is the right choice when the same ECC
checkout is also used from Claude Code, because both harnesses then share the
learned state.

Verified with A: `session-data/`, `metrics/tool-usage.jsonl`, `metrics/costs.jsonl`,
`homunculus/projects/` and `gateguard/state-session-<id>.json` all appear under
`$DSH_HOME/ecc/state`, and every probed hook exits 0. One residual message
remains in either mode: the Stop hook `continuous-learning` reaches for a fixed
`~/.claude/skills/learned` (`getLearnedSkillsDir` in `scripts/lib/utils.js`)
instead of an overridable root.

## Scope

| Surface | Status |
|---------|--------|
| Skills | Whole `skills/` catalog, every profile, no symlinks |
| Hooks | Bridged events only: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart`, `SubagentStop`. `PreCompact`, `PostToolUseFailure`, `SessionEnd` are dropped by the generator |
| Commands | Not available: DSH has no Claude Code slash-command surface; the workflows stay reachable through skills |
| Agents | ECC subagent definitions are not mounted; DSH agent presets are the native equivalent |

## Uninstall

Remove `ecc-dsh` from `dsh.profile.bundles` (or disable the two rows in the
profile's `cordis.patch.yml`), then delete `~/.dsh/ecc/hooks-dsh.json`.

## Source

Developed in the [ECC](https://github.com/affaan-m/ECC) checkout under `plugins/dsh/`
and published here for path or git-spec installs. ECC is MIT-licensed (Copyright (c)
2026 Affaan Mustafa); the license travels with this repository.
