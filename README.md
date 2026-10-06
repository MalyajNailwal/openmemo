# OpenMemo

Permanent, local-first memory for AI coding agents. OpenMemo is a hardened,
backward-compatible fork of [VictorTaelin/OptMem](https://github.com/VictorTaelin/OptMem):
one Python CLI, an append-only text log, a binary summary tree, and no required
dependencies or background service.

> **Attribution and license notice:** all original design credit belongs to
> Victor Taelin and the OptMem contributors. This repository does not currently
> have a resolved upstream license. Clarify the license before publishing,
> redistributing, or offering this fork as a service.

[![Tests](https://github.com/MalyajNailwal/openmemo/actions/workflows/tests.yml/badge.svg)](https://github.com/MalyajNailwal/openmemo/actions/workflows/tests.yml)

![How the OptMem merge tree works](anim/optmem.gif)

## Why this fork exists

The original storage model stays intact, while this fork adds safer handling of
untrusted memories, concurrent writers, optional decision-model assistance,
cross-platform tests, and clearer operational guidance.

| Area | Changes in this fork |
|---|---|
| Trust boundary | `wake`, `recall`, `zoom`, and compression inputs frame memories as untrusted data; raw notes are JSON-quoted |
| Concurrent writes | Local file locking serializes notes on POSIX and native Windows; IDs are assigned while the lock is held |
| Crash handling | A partial trailing fixed-width record is repaired before the next append |
| Memory quality | Optional timestamps, source, tags, importance, contradiction hints, and injection quarantine live in append-only `META.jsonl` |
| Search | Streaming regex recall remains the default; optional local candidate selection and Jev reranking are available with `--smart` |
| Privacy | Credential patterns block external classification; email addresses and phone numbers are redacted before a Jev request |
| Optional AI | TypeSafe direct, Vercel TypeSafe-compatible, and Vercel native Jev backends; disabled by default and fail-open |
| Portability | Standard-library-only Python; CI runs the core and mocked-network suites on Linux, macOS, and Windows |
| Compatibility | Existing `LOG.txt` and `TREE/` stores work without migration; the 426-token agent prompt is unchanged |

See [CHANGELOG.md](CHANGELOG.md) for the release-oriented list and
[SECURITY.md](SECURITY.md) for the trust model and reporting guidance.

## Core principles

- Plain-text, human-inspectable, append-only source of truth.
- One Python file and zero required dependencies.
- Nothing runs in the background.
- Local behavior remains fully usable without an API key or network.
- Optional integrations must fail open and must not modify old log records.

## Requirements

- Python 3.7 or newer.
- A shell for the commands below. Native Windows instructions are in
  [WINDOWS.md](WINDOWS.md).
- Jev is optional; normal memory operations need no account or API key.

## Install this fork

Installing from a reviewed clone avoids piping a mutable remote script directly
into a shell:

```sh
git clone https://github.com/MalyajNailwal/openmemo.git
cd openmemo
mkdir -p "$HOME/.optmem"
cp memo "$HOME/.optmem/memo"
chmod 700 "$HOME/.optmem"
chmod 755 "$HOME/.optmem/memo"
"$HOME/.optmem/memo" init
```

`init` prints a `## Memory` block. Paste it at the top of the agent's
`AGENTS.md` or `CLAUDE.md`. Add `~/.optmem` to `PATH` if you want to invoke the
tool as `memo` instead of `~/.optmem/memo`.

To update the executable after reviewing a newer checkout:

```sh
cp memo "$HOME/.optmem/memo"
chmod 755 "$HOME/.optmem/memo"
```

Updating the CLI does not rewrite `LOG.txt`. Existing OptMem stores require no
migration for this fork.

## Quick start

```sh
memo init
memo note "The deployment region is ap-south-1."
memo wake
memo recall 'deployment|region'
```

Run `memo wake` at the start of an agent session. `note` may print one pending
compression task; complete it with the exact `memo nap ...` command it provides.
No scheduler or daemon performs merges automatically.

## Commands

| Command | Purpose |
|---|---|
| `memo init` | Create a memory store and print the agent prompt block; safe to rerun |
| `memo wake [part [T]]` | Read the bounded memory context, with stable pagination |
| `memo note "..."` | Append one UTF-8 memory line, up to 280 bytes by default |
| `memo nap [<lo>-<hi> "..."]` | Show or save the next merge-tree summary |
| `memo recall <regex>` | Case-insensitive regex search over the complete log |
| `memo recall --smart <query>` | Select candidates locally, then optionally rerank them with Jev |
| `memo recall --all <regex>` | Reveal quarantined text during deliberate inspection |
| `memo zoom <lo>-<hi>` | Expand one summary node into its two children |
| `memo forget <lo>-<hi>` | Remove a bad cached summary so `nap` can rebuild it; never deletes log entries |
| `memo config [NAME=VALUE]` | Display or change reading and record-size limits |
| `memo import <file>` | Bootstrap dated entries from `YYYY-MM-DD <text>` lines |
| `memo jev-check [--dry-run]` | Check optional Jev configuration using synthetic text only |

Run `memo` without arguments to print the built-in command summary. Invalid
commands and arguments return a concise usage error.

## How storage works

```text
~/.optmem/
├── memo              # single-file Python CLI
└── memory/
    ├── LOG.txt       # authoritative append-only memories
    ├── TREE/         # rebuildable fixed-width summary records
    ├── META.jsonl    # optional append-only metadata events
    ├── config        # per-store size overrides
    └── .lock         # local inter-process coordination
```

- Each `LOG.txt` record is exactly 320 bytes. Its byte offset determines its
  immutable ID, so direct lookup takes one seek.
- Note text is limited by UTF-8 **bytes**, not Unicode character count. The
  default payload limit is 280 bytes.
- Tree records are 288 bytes. Aligned power-of-two ranges (`#0-1`, `#0-3`, and
  so on) form a binary merge tree over the log.
- Recent memories remain detailed. Older ranges are represented by bounded
  summaries so `wake` has a stable reading budget instead of an ever-growing
  prompt.
- `TREE/` is a cache: `forget` can discard a bad summary and `nap` rebuilds it
  without touching the log.
- `META.jsonl` is supplemental. Invalid metadata is warned about and ignored;
  it never makes the authoritative log unreadable.

### Store location and configuration

Set `MEMORY_DIR` to keep the store somewhere other than
`~/.optmem/memory`:

```sh
export MEMORY_DIR="$PWD/.memory"
memo init
```

Use a temporary `MEMORY_DIR` for tests and experiments. A typo does not create
a second identity: only `memo init` is allowed to create a new store.

```sh
memo config                  # display all size settings
memo config WAKE_LINES=300   # change the wake reading budget
memo config WAKE_LINES=      # restore the default
```

`WAKE_LINES` affects only what is rendered; changing it does not rewrite or
recompute stored memories.

Local locking protects multiple processes on one machine. It cannot coordinate
two machines concurrently writing the same cloud-synced or Git-backed folder.
Back up `LOG.txt` and `META.jsonl`, and ensure only one machine writes a shared
store at a time.

## Safety and privacy

Memory can originate from users, websites, command output, or imported files.
It is data, not authority. The CLI places memory output between explicit
untrusted-data markers and JSON-quotes raw notes. Likely instruction injection
can also be quarantined when the optional Jev hook is enabled. These are
defense-in-depth controls, not a proof that prompt injection is impossible.

The store is not encrypted. Do not record passwords, API keys, access tokens,
private keys, recovery codes, or other credentials. Restrict filesystem access,
protect backups, and use full-disk encryption where appropriate.

## Optional Jev decisions

[Jev](https://docs.typesafe.ai/introduction) is TypeSafe AI's System One
decision model. It returns typed choices, scores, boolean probabilities, and
confidence; it does not generate note or summary prose. This integration uses
only Python's `urllib` and `json` modules.

Jev is **off by default**. Both the master switch and an individual feature
switch must be enabled:

```sh
export OPTMEM_JEV=1
export OPTMEM_JEV_KEY='your-key'       # never save this in memory or the repo
export OPTMEM_JEV_NOTE_GATE=1          # long-term-value gate
export OPTMEM_JEV_SUPERSEDE=1          # duplicate/contradiction hints
export OPTMEM_JEV_TAGS=1               # bounded tag and importance metadata
export OPTMEM_JEV_INJECTION=1          # second-opinion quarantine signal
export OPTMEM_JEV_SMART_RECALL=1       # AI reranking for recall --smart
```

Create a TypeSafe key in the [TypeSafe console](https://console.typesafe.ai/)
or an AI Gateway key in the Vercel dashboard. Never place the key in
`LOG.txt`, `META.jsonl`, the repository, or checked-in shell configuration.

### Providers

| Backend | Base URL | Model | API behavior |
|---|---|---|---|
| `typesafe` | `https://api.typesafe.ai` | `jev-latest` | TypeSafe `/v1/systemone`; full typed answers and confidence |
| `vercel-compatible` | `https://ai-gateway.vercel.sh/typesafe` | `typesafe-ai/jev` | TypeSafe-compatible `/v1/systemone`; recommended Vercel mode |
| `vercel-native` | `https://ai-gateway.vercel.sh` | `typesafe-ai/jev` | Native `/v1/evaluate` boolean Decision API |
| `none` | — | — | Force all Jev behavior off |

TypeSafe direct example:

```sh
export OPTMEM_JEV_BACKEND=typesafe
export OPTMEM_JEV_BASE_URL=https://api.typesafe.ai
export OPTMEM_JEV_MODEL=jev-latest
```

Vercel TypeSafe-compatible example:

```sh
export OPTMEM_JEV_BACKEND=vercel-compatible
export OPTMEM_JEV_BASE_URL=https://ai-gateway.vercel.sh/typesafe
export OPTMEM_JEV_MODEL=typesafe-ai/jev
```

The implementation follows the official [TypeSafe introduction](https://docs.typesafe.ai/introduction),
[Vercel TypeSafe-compatible API](https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe),
[Vercel evaluation API](https://vercel.com/docs/ai-gateway/modalities/evaluation),
and [Jev gateway announcement](https://vercel.com/changelog/ai-gateway-now-supports-typesafe-clients-and-an-http-api-for-jev).

### Jev settings

| Variable | Default | Meaning |
|---|---:|---|
| `OPTMEM_JEV` | off | Master switch |
| `OPTMEM_JEV_BACKEND` | `typesafe` | `typesafe`, `vercel-compatible`, `vercel-native`, or `none` |
| `OPTMEM_JEV_KEY` | unset | Provider API key |
| `OPTMEM_JEV_BASE_URL` | provider default | Override the provider base URL |
| `OPTMEM_JEV_MODEL` | provider default | Override the model name |
| `OPTMEM_JEV_TIMEOUT` | `3.0` | HTTP timeout in seconds, from 0.1 to 60 |
| `OPTMEM_JEV_TOP_N` | `5` | Local candidates considered, from 1 to 20 |
| `OPTMEM_JEV_NOTE_THRESHOLD` | `0.60` | Minimum long-term-value probability required to save |
| `OPTMEM_JEV_CONTRADICTION_THRESHOLD` | `0.80` | Minimum relationship probability to surface a candidate |
| `OPTMEM_JEV_INJECTION_THRESHOLD` | `0.90` | Probability at which a note is quarantined |
| `OPTMEM_JEV_MIN_CONFIDENCE` | `0.50` | Minimum confidence for choice/score actions |
| `OPTMEM_JEV_DRY_RUN` | off | Print the redacted request without sending it |

Per-command controls:

```sh
memo note --no-ai "Always keep this operation local."
memo note --dry-run "Show the redacted Jev payload, but still save locally."
memo recall --smart --no-ai 'deployment region'
memo recall --all 'quarantined text'
memo jev-check --dry-run
```

API timeout, network failure, missing key, and malformed response all fail open:
the normal local operation continues with a short warning. A valid note-gate
result below the configured threshold deliberately skips that new note.
Contradiction results are advisory and never rewrite or delete history.

### Third-party data and cost

When a Jev feature is active, note text or locally selected recall candidates
are sent to TypeSafe AI or Vercel. Before request construction, a local
preflight blocks common credential patterns and redacts email addresses and
phone numbers. Pattern filters cannot guarantee removal of every secret or
identifier. Keep Jev disabled—or use `--no-ai`—for content that must stay
entirely local.

As of October 2026, Vercel lists `typesafe-ai/jev` at $0.04 per million input
tokens and publishes no numeric latency guarantee. Pricing can change; verify
the [live Jev model page](https://vercel.com/ai-gateway/models/jev). Disable the
integration by unsetting `OPTMEM_JEV`, setting it to `0`, or selecting backend
`none`.

```sh
memo jev-check              # sends one fixed synthetic sentence
memo jev-check --dry-run    # sends nothing
```

## Testing

All tests use temporary memory directories. Jev HTTP calls are mocked; the
test suite needs no API key and sends no memory over the network.

```sh
python3 test.py
python3 test_jev.py
```

GitHub Actions runs both suites on Linux, macOS, and Windows. Coverage includes
UTF-8 record boundaries, damaged records, concurrent notes, disabled/failing
Jev, malformed responses, secret filtering, redaction, thresholds, dry-run,
and smart recall.

## Known limitations

- Regex recall is exact/local by default; Jev reranking considers only locally
  selected candidates and is not semantic vector search.
- Metadata can mark a contradiction or quarantine a note, but `LOG.txt` remains
  immutable and there is no automatic destructive supersede operation.
- Local file locking does not make a cloud-synced directory a distributed
  database.
- Pattern-based secret and injection detection can have false positives and
  false negatives.
- Memory content and metadata are not encrypted at rest.
- The unresolved upstream license blocks safe public redistribution until the
  owner clarifies it.

## Project documents

- [CHANGELOG.md](CHANGELOG.md) — changes introduced by this fork.
- [SECURITY.md](SECURITY.md) — security boundaries and private reporting.
- [WINDOWS.md](WINDOWS.md) — native Windows usage and locking behavior.
- [Original OptMem](https://github.com/VictorTaelin/OptMem) — upstream project
  and design credit.

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
