# Syntato fork — what's different from upstream, and why

This is a fork of [LibreChat-AI/code-interpreter](https://github.com/LibreChat-AI/code-interpreter)
(also mirrored at `ClickHouse/code-interpreter`, which is where LibreChat's
own docs point self-hosters — that's where this fork was originally cloned
from). It backs Code Interpreter for `lab-agent`, Syntato's LibreChat
deployment — see that repo's README for the full picture.

Kept as a real fork (not a private mirror) specifically so a clean upstream
PR/issue stays possible later — see "Reporting upstream" below.

Everything here is on top of upstream `main`, in `docker-compose.local-dev.yml`
and `api/Dockerfile` only. Nothing in `package-init.sh` or any other core
file is touched — deliberately, so this stays a small, low-conflict diff
against a project whose core files (the package list especially) keep
changing upstream. Check `git log origin/main..main` for the exact
commits; this file explains the *why*, the diff itself is the *what*.

## 1. `docker-compose.local-dev.yml`

### `CODEAPI_INTERNAL_SERVICE_TOKEN` left unset

This file's own header says "Local Development Stack - No authentication
required," but ships a non-empty token default anyway, which switches
internal-service auth **on**. With it on, every real job that primes a
skill's bundled file into the sandbox 401'd — traced by capturing raw
traffic between the `sandbox` and `file_server` containers: the sandbox's
actual runtime (`Bun`) never attaches the `X-CodeAPI-Internal-Token` header
`file_server` requires, no matter what the token is set to. A manual `curl`
with the header set always worked; the real job never did.

Leaving the token unset makes `requireInternalServiceAuth` skip the check
entirely (see `internal-service-auth.ts`), which is what this file already
claims to be. This is the one change here that's a real upstream bug, not
just a local preference — see "Reporting upstream."

### `file_server` host port remapped 3000 → 3010

Purely local: host port 3000 on this machine is already owned by an
unrelated LibreChat-ecosystem container (`admin-panel`, part of
`lab-agent`). Nothing to do with this project.

### `restart: unless-stopped` added to all six services

Missing from the file as shipped. Without it, the whole stack silently died
the first time this host slept and stayed dead — nothing brought it back.
Not a bug exactly (this is explicitly a "local dev" compose file), but
worth having for anything left running unattended.

## 2. `api/Dockerfile` — biopython baked into the sandbox

The `biopython` skill (see `lab-agent-skills`) needs the actual `Bio`
package to run real code (`Seq.translate()`, alignments, PDB parsing, …),
not just documentation the model reasons from. It wasn't in
`package-init.sh`'s baked package list, and the sandbox has zero network
egress (`SANDBOX_DISABLE_NETWORKING=true`) — so there is no way to `pip
install` it at request time, baking it into the image at build time is the
only option.

Added as its own `RUN` layer immediately after the existing package-builder
step, rather than editing `package-init.sh`'s hardcoded package array in
place — same reasoning as above: that array is the part of this project
most likely to keep changing upstream, so touching it directly would be the
most conflict-prone possible place to diverge. One isolated added line is
much cheaper to carry forward through a future `git pull`/rebase from
upstream.

Pinned to `biopython==1.87` to match what the skill's own `SKILL.md`
documents itself as targeting. Verified directly against a running
sandbox: `Bio.__version__ == '1.87'`, `Seq.translate()` produces correct
output.

**Known limitation, not fixed by this:** baking the package gets local
computation working (translation, alignment, parsing files already on
disk), but not `Bio.Entrez` (NCBI/PubMed/GenBank queries) or remote BLAST —
those need live internet access at request time, and no amount of baking
in fixes that without actually enabling sandbox networking, which is a much
bigger decision (see below).

## Considered and rejected: `SANDBOX_DISABLE_NETWORKING=false`

Real flag, actually exists (`api/src/config.ts`) — worth recording that it
was seriously considered, not overlooked. Checked whether it (or the
"egress gateway" subsystem, which sounded promising) offers anything
*scoped* — e.g. "allow network to pypi.org only." It doesn't:
`egress-gateway.ts`/`egress-grant.ts`/`egress-ledger.ts` mediate and audit
the sandbox's calls to *our own internal services* (file downloads,
programmatic tool-call callbacks) in the fully-hardened production mode —
unrelated to general internet access. `SANDBOX_ALLOWED_LOCAL_NETWORK_PORT`/
`SANDBOX_FORWARD_TARGET` is a single fixed internal port-forward, also not
applicable to external domains. There is no allowlist mechanism anywhere in
this codebase — flipping `SANDBOX_DISABLE_NETWORKING` is genuinely
all-or-nothing: full outbound internet access for *any* code *any*
conversation ever runs, forever, not just pip installs.

Given a working, verified, zero-exposure fix already exists (baking the
package in), that tradeoff isn't worth it just to unblock `pip install
biopython`. Worth revisiting only if/when `Bio.Entrez`-style live-internet
features are actually wanted — that's a bigger, separate decision, not a
follow-on to this one.

## Reporting upstream

The `CODEAPI_INTERNAL_SERVICE_TOKEN`/missing-header bug is real and would
hit anyone else self-hosting with a token set — worth an issue against
[LibreChat-AI/code-interpreter](https://github.com/LibreChat-AI/code-interpreter)
(the actual root of this fork network, not `ClickHouse/code-interpreter`).
Not yet filed. What we have is a workaround (disable the auth check
entirely), not a fix to the actual bug (Bun not attaching the header) — filing
an issue describing the repro is realistic; a PR would need someone to
actually debug and patch the Bun-side request code, which hasn't happened.

The port remap and restart-policy additions aren't upstream material —
one's host-specific, the other's a minor, easy local addition not worth a
PR's worth of ceremony.
