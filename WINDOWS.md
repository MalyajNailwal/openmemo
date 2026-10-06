# Windows support

OptMem now runs on native Windows (no WSL required).

## What changed
- `import fcntl` is guarded — falls back to `None` on platforms without it.
- `locked()` uses `msvcrt` advisory locking with spin/backoff when `fcntl`
  is unavailable, so parallel sessions (the documented multi-process case)
  queue instead of raising `Resource deadlock avoided`.
- The `.lock` file is opened in append mode (`"a"`) rather than `"w"`, which
  would truncate and break locks held by other processes on Windows.

## Test (Windows native, no WSL)
```bat
set MEMORY_DIR=C:\path\to\mem
python memo init
python memo note "first memory"
python memo note "second memory"
python memo wake
```
Concurrency: 8 parallel `memo note` processes writing 1600 memories
resulted in 1600/1600 records persisted (lock verified).

## Optional Jev

Jev remains off unless explicitly enabled. In `cmd.exe`:

```bat
set OPTMEM_JEV=1
set OPTMEM_JEV_BACKEND=typesafe
set OPTMEM_JEV_KEY=your-key
set OPTMEM_JEV_NOTE_GATE=1
python memo jev-check
```

Use `set OPTMEM_JEV=0` to disable it. Never paste the key into a note, checked-in
configuration, or diagnostic output. See the README for Vercel AI Gateway and
privacy configuration.
