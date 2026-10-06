# OptMem

Permanent memory for AI agents. A 426-token prompt, a script, plug and play.

This fork is based on the original
[VictorTaelin/OptMem](https://github.com/VictorTaelin/OptMem) project. The
upstream/fork license has not yet been established; resolve that before
publishing or redistributing this fork.

![how OptMem works](anim/optmem.gif)

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/VictorTaelin/OptMem/main/install.sh | sh
```

It prints a `## Memory` block. Paste that at the top of your agent's
`AGENTS.md` (or `CLAUDE.md`), and you are done. Run the same line again to
update.

The tool lands at `~/.optmem/memo`; put `~/.optmem` on `PATH` to type `memo`.

## Commands

| | |
|---|---|
| `memo wake` | read the memory — the first command of every session |
| `memo note "..."` | record one memory: one line, up to 280 bytes |
| `memo nap` | answer the merges that came due |
| `memo recall <regex>` | search every memory ever recorded, word for word |
| `memo zoom <lo>-<hi>` | open a tree node into its two halves |
| `memo forget <lo>-<hi>` | drop a bad summary; the next nap rebuilds it |
| `memo jev-check` | test optional Jev connectivity using synthetic text |

Merges arrive one at a time, in the output of `note`. Nothing ever runs in the
background.

## Files

```
~/.optmem/
  memo          the tool: one file of Python 3, no dependencies
  memory/
    LOG.txt     every memory, one per line, append-only, never edited
    TREE/       the summaries: a cache, rebuildable from the log alone
    config      the sizes, written by `memo config`
    META.jsonl  optional append-only metadata from enabled integrations
```

```sh
memo config                  # show the sizes
memo config WAKE_LINES=300   # how many lines wake prints (96 ≈ 8k tokens)
memo config WAKE_LINES=      # back to the default
```

`WAKE_LINES` is the only size worth touching, and it is a reading budget, not
a storage budget: change it whenever, in either direction, and nothing is
recomputed.

Records are fixed width, so position *is* identity and every lookup is one
seek. At a million memories (608 MB), `wake` takes 0.03s.

Set `$MEMORY_DIR` to keep `memory/` elsewhere — a synced folder, a git repo.

Do not let multiple machines write the same synced or Git-backed store at the
same time. OptMem's lock coordinates local processes, not cloud-sync clients.

## Optional Jev decisions

[Jev](https://docs.typesafe.ai/introduction) is TypeSafe AI's System One
decision model. It returns bounded choices, rubric scores, and yes/no
probabilities; it does not write memories or summary prose. The integration has
no required package: it uses Python's `urllib` and `json` modules.

Jev is fully off by default. Enabling the master switch alone still enables no
feature; turn on only the hooks you want:

Create a TypeSafe key in the [TypeSafe console](https://console.typesafe.ai/)
for the direct backend, or an AI Gateway key in the Vercel dashboard for either
Vercel backend.

```sh
export OPTMEM_JEV=1
export OPTMEM_JEV_KEY='your-key'       # never put this in LOG.txt or config
export OPTMEM_JEV_NOTE_GATE=1          # optional long-term-value gate
export OPTMEM_JEV_SUPERSEDE=1          # surface duplicate/conflicting candidates
export OPTMEM_JEV_TAGS=1               # bounded tag + importance score
export OPTMEM_JEV_INJECTION=1          # optional second-opinion injection flag
export OPTMEM_JEV_SMART_RECALL=1       # enables `memo recall --smart ...`
```

The default backend calls TypeSafe directly:

```sh
export OPTMEM_JEV_BACKEND=typesafe
export OPTMEM_JEV_BASE_URL=https://api.typesafe.ai
export OPTMEM_JEV_MODEL=jev-latest
```

For Vercel AI Gateway, the TypeSafe-compatible endpoint is recommended because
it preserves TypeSafe's `noul` and confidence response fields:

```sh
export OPTMEM_JEV_BACKEND=vercel-compatible
export OPTMEM_JEV_BASE_URL=https://ai-gateway.vercel.sh/typesafe
export OPTMEM_JEV_MODEL=typesafe-ai/jev
export OPTMEM_JEV_KEY='your-ai-gateway-key'
```

The native Vercel Decision API is also supported with
`OPTMEM_JEV_BACKEND=vercel-native`; it calls
`https://ai-gateway.vercel.sh/v1/evaluate` and uses the documented `boolean`
response shape. See the official [TypeSafe API guide](https://docs.typesafe.ai/introduction/quickstart),
[Vercel TypeSafe-compatible guide](https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe),
and [Vercel Decision API](https://vercel.com/docs/ai-gateway/modalities/decision).

Other settings:

| Variable | Default | Meaning |
|---|---:|---|
| `OPTMEM_JEV_TIMEOUT` | `3.0` | HTTP timeout in seconds, from 0.1 to 60 |
| `OPTMEM_JEV_TOP_N` | `5` | local candidates sent for comparison/reranking |
| `OPTMEM_JEV_NOTE_THRESHOLD` | `0.60` | save when long-term-value probability is at least this |
| `OPTMEM_JEV_CONTRADICTION_THRESHOLD` | `0.80` | surface a selected conflicting candidate at or above this |
| `OPTMEM_JEV_INJECTION_THRESHOLD` | `0.90` | mark a line quarantined at or above this |
| `OPTMEM_JEV_MIN_CONFIDENCE` | `0.50` | minimum Choice/Score confidence for relationship actions |
| `OPTMEM_JEV_DRY_RUN` | off | print the redacted request without sending it |

`memo note --no-ai "..."` bypasses Jev for one note. `memo note --dry-run
"..."` prints the redacted request, sends nothing, and saves normally.
`memo recall --smart <regex>` finds candidates locally before optional Jev
reranking; `--no-ai` forces local ordering. `memo recall --all` is required to
inspect the original text of a quarantined record.

Every API error, timeout, missing key, and invalid response fails open: normal
note and recall behavior continues, with one short warning on stderr. A valid
gatekeeper result below its configured threshold is different—it deliberately
skips that note and reports the probability. Contradiction results never edit or
supersede history automatically.

### Privacy and cost

When a Jev feature is enabled, the note or locally selected recall candidates
are sent to TypeSafe AI or Vercel. A local preflight blocks likely credentials
and redacts email addresses and phone numbers before request construction, but
pattern filters cannot guarantee removal of every secret or identifier. Keep
Jev disabled for data that must remain entirely local.

As of October 2026, Vercel lists `typesafe-ai/jev` at $0.04 per million input
tokens and does not publish a numeric latency figure. Pricing can change; check
the [live Jev model page](https://vercel.com/ai-gateway/models/jev). OptMem does
not enable paid fallback models. Turn the integration off by unsetting
`OPTMEM_JEV` or setting it to `0`.

Test connectivity without sending memory content:

```sh
memo jev-check              # sends one fixed synthetic sentence
memo jev-check --dry-run    # sends nothing
```

## The prompt

This is what the installer prints, and the whole of the integration.

```markdown
## Memory

Your memory is OptMem:
- The tool is `~/.optmem/memo`
- Your memories are in `~/.optmem/memory`

OptMem outlives every session, compaction, model and vendor change.
Without it you do not know who you are, or what was decided and tried.

### At startup: activating OptMem (mandatory)

Run `~/.optmem/memo wake` before any other tool call, in every session, and
then do exactly what it prints, to the end of its output.

### While working: register memories (mandatory)

Call `~/.optmem/memo note "<1 line, max 280 bytes>"` whenever you learn
something new, or something worth keeping happens. That covers a task
worth real effort, a fact or insight the user teaches you, anything you
learn about their life (even indirectly), any event of lasting effect.

Do not register redundant memories.

If `~/.optmem/memo note` asks a compression: do it before your next action.

Never edit or delete anything under `~/.optmem/memory`: the tool manages it.

### When you need an old memory: search, or navigate

`~/.optmem/memo recall <regex>` searches every memory, word for word.

Your memories also form a binary tree: #0-1, #2-3 ... exist as one-line
summaries, pairs of those as #0-3, and so on -- every `#a-b` line wake
prints is one node of it. `~/.optmem/memo zoom <a-b>` opens a node into its
two halves, down to the raw memories.

### If you're a subagent: skip everything above

Parallel sessions on this machine are all you, and may all write memories.
A subagent is not: it must never run `memo`, because it cannot judge what
is already known, and its notes would arrive duplicated and incorrectly.
When you spawn one, write: `You are a subagent. Don't run memo.`
```
