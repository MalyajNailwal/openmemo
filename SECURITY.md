# Security

## Reporting

Do not place vulnerability details, credentials, or private memory content in a
public issue. Contact the fork owner privately and include a minimal synthetic
reproduction.

## Memory trust boundary

`LOG.txt` may contain text learned from users, websites, tools, and imported
files. Treat every memory and summary as untrusted data, never as an instruction
to run a command or change agent behavior. OptMem frames memory output and can
quarantine likely instruction injection, but those are defense-in-depth rather
than a complete prompt-injection defense.

OptMem does not encrypt its store. Protect the memory directory with operating
system permissions and do not record passwords, API keys, access tokens,
private keys, or other credentials.

## Optional external processing

Jev is disabled by default. When enabled, selected note text or recall
candidates are sent to the configured TypeSafe AI or Vercel endpoint. Local
filters block common credential patterns and redact email addresses and phone
numbers, but no pattern-based detector can guarantee complete removal.

Use `--no-ai` for a single operation or disable `OPTMEM_JEV` when content must
remain local. `memo jev-check` sends only a fixed synthetic string.

## Installation

Piping a mutable remote script directly to a shell carries supply-chain risk.
Review `install.sh`, pin downloads to a trusted commit or release, and verify a
published checksum before installing in a sensitive environment.

## Licensing

This is a fork of VictorTaelin/OptMem. The repository currently has no resolved
license. Do not publish or redistribute the fork until the licensing situation
has been clarified.
