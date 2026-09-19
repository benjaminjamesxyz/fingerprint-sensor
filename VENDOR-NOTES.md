# Vendor notes — local deviations from upstream cityji/honor-magickbookpro-fingerprint-driver

Everything here is upstream code except `polkit/50-fpc-a900.rules` (ours) and
`packaging/` (ours). Upstream's README is preserved verbatim at
`docs/UPSTREAM-README.md`; the top-level `README.md` is a fresh setup guide.
Keep this file accurate so future upstream diffs stay reviewable.

## `.github/workflows/ci.yml`, `.github/workflows/release.yml`

Cosmetic only: long lines rewrapped to satisfy a local 80-column lint gate
(shell continuations, `$mr`/`$f`/`$url` shell variables). Every command is
byte-for-byte equivalent in effect; rendered release notes are unchanged.
No job was added, removed, or reordered.

## `.github/workflows/*` — security hardening (functional, deliberate)

In addition to the rewraps above, CI was hardened against supply-chain
findings (zizmor/semgrep): actions pinned to full commit SHAs
(checkout v4.4.0, upload-artifact v4.6.2, download-artifact v4.3.0),
`persist-credentials: false` on every checkout, and the release token
scoped to the `publish` job (`contents: write` there, `read` everywhere
else). Runtime behaviour is otherwise unchanged.

## `tools/probe-blob.c` — `_IONBF` (line 8)

A local lint gate flags `_IONBF` as a reserved-namespace identifier. False positive:
`_IONBF` is the POSIX-mandated macro from `<stdio.h>` (`setvbuf` buffering
mode, alongside `_IOFBF`/`_IOLBF`). It is *supposed* to live in the
implementation namespace — the standard reserves those names precisely so
the implementation can define it. The identifier is not ours to rename;
using it is the only correct spelling. Upstream line 7; locally shifted to
line 8 by the one-line `// pi-lens-ignore: reserved-identifier` comment
above it, recording the triage decision in-tree. Do not "fix" either the
code or the comment away.
