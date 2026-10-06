# Changelog

## Unreleased

### Added

- Optional, disabled-by-default Jev integration using only Python's standard
  library.
- TypeSafe direct, Vercel TypeSafe-compatible, and Vercel native Decision API
  adapters with strict typed-response validation.
- Opt-in note value gating, contradiction surfacing, bounded auto-tags,
  importance scoring, instruction-injection quarantine, and smart recall.
- Local credential/PII preflight, `--no-ai`, redacted dry-run mode, and
  fail-open behavior.
- Append-only `META.jsonl` for optional tags and classification metadata while
  preserving the existing `LOG.txt` format.
- `memo jev-check`, which tests connectivity using only synthetic content.
- Explicit untrusted-data framing for memory output.
- Offline mocked Jev tests and a cross-platform GitHub Actions workflow.

### Changed

- `wake`, `recall`, `zoom`, and compression prompts now identify memory-derived
  content as untrusted data.
- Quarantined records remain recoverable with `memo recall --all` but their
  original text is hidden from normal wake output.

### Compatibility

- Existing `LOG.txt` and `TREE/` files require no migration.
- Jev is off unless both `OPTMEM_JEV=1` and an individual feature flag are set.
- The agent prompt block is unchanged.

### Attribution and publishing notice

- This fork credits the original VictorTaelin/OptMem project.
- The upstream/fork license remains unresolved and must be clarified before
  publication or redistribution.
