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

## 3. `api/Dockerfile` — pydna baked into the sandbox

Same reasoning as biopython above, for a skill covering molecular cloning
design (restriction digestion/ligation simulation, Type IIS-flanked
construct design) rather than sequence analysis. pydna's own dependency
list already includes `biopython` (baked in above) plus `networkx`,
`numpy`, `appdirs`, `prettytable`, and a few smaller pure-Python packages —
all local computation, nothing that needs live network access for the
digestion/ligation/assembly-simulation functionality actually used here.

Notably, pydna also depends on `opencloning-linkml` — the two projects
(pydna, OpenCloning) share a data model; OpenCloning's own automation is
built on pydna as its simulation engine, not a separate implementation.
Building on pydna locally gets the same simulation fidelity without
needing OpenCloning's hosted service or its network dependency.

License checked before adding: `LICENSE.txt` in the pydna repo is a
BSD-3-Clause-style permissive license (GitHub's own detector just doesn't
recognize it as a template match, since the org/copyright names are
substituted into the license text itself) — properly licensed, unlike a
previously-considered alternative for BLAST tooling that was rejected
specifically for having no LICENSE file at all.

Pinned to `pydna==5.5.16` (the latest release at the time) to match what
the skill's own `SKILL.md` documents itself as targeting.

## 4. Forward FASTA output files to lab-agent-preview-server

**This touches core files, not just `docker-compose.local-dev.yml`/
`api/Dockerfile`** — everything above this section stayed off core files
deliberately, to keep this fork's diff small and low-conflict. Ended up
touching three here, each forced by a real, only-discovered-by-testing
constraint (not assumed up front — the actual diff grew twice past the
original one-file estimate while getting this working end to end):

1. **`api/src/job.ts`** — `SUPPORTED_EXTENSIONS` (the allowlist deciding
   which sandbox output files even get uploaded at all) is a hardcoded
   `Set` literal with no env var or config knob. Added `.fasta`/`.fa`,
   plus one new best-effort private method (`forwardToPreviewServer`,
   called once from the end of the existing `uploadOneFile`). No changes
   to `FileRef`, `ExecuteResult`, `TFile`, or any response-shaping code
   in `api/src/api/v2.ts` — see "Not done" below for why.
2. **`launcher/src/main.rs`** — `job.ts` doesn't run as a plain container
   process; it's the guest's top-level process inside a nested libkrun
   microVM (confirmed via `docker exec sandbox` showing only a
   `VM:<hash>` process, no bare `bun`/`node`). `is_allowed_guest_env_key()`
   is an explicit allowlist of which container env vars even get
   forwarded into that guest at all — `FILE_SERVER_URL` was already on it
   (`LEGACY_NON_EGRESS_EXACT`), `PREVIEW_SERVER_URL` wasn't, so it was
   silently invisible to `job.ts` until added there too.
3. **`launcher/entrypoint.sh`** — libkrun's TSI (Transparent Socket
   Impersonation) networking proxies the guest's socket syscalls into the
   container's own netns transparently, so *reachability* was never the
   issue (confirmed: `docker exec sandbox curl http://host.docker.internal:8012`
   already worked). The actual gap: TSI can't do the guest's own DNS
   (`launcher/entrypoint.sh`'s own header comment already said so, for
   `FILE_SERVER_URL`'s sake), so `resolve_url` pre-resolves configured
   hostnames to literal IPs in the container's namespace *before* the
   guest boots. `PREVIEW_SERVER_URL` needed the same one-line treatment
   `FILE_SERVER_URL` already gets, or the guest saw an unresolvable
   hostname string and every forward attempt failed with a
   resolution-stage `ConnectionRefused`.

Correction to an earlier draft of this entry: it claimed `job.ts` runs
"not from inside the nsjail/microVM guest" — wrong, confirmed by the
investigation above. What's actually true, and what actually matters for
"no sandbox networking change needed": `SANDBOX_DISABLE_NETWORKING`/the
nsjail seccomp network-syscall filter (`api/src/nsjail.ts:167-170`) is
scoped to the nsjail'd child process that runs **user-submitted code**
specifically, not to `job.ts` itself — `job.ts`'s own outbound calls
(file_server, now preview-server) were never inside that boundary. That
distinction is real and load-bearing; the container-vs-guest one was not.

**Why this exists**: `lab-agent-preview-server` gives the agent a way to
show a human a rendered FASTA file — but the only way in was
`preview_fasta(content=...)`, an MCP tool call, meaning the agent had to
retype every byte of the file as a string argument. Asked for a 50+
sequence file, it gave up partway through and reported false success —
the stored preview really only had 5 records plus a literal `...` it
wrote itself. Writing the file via code execution and having the output-
file extraction pipeline carry it out sidesteps that: the sequence data
never has to pass through the model's own generated text.

**What was actually missing**, confirmed by reading the code rather than
assumed: `.fasta` was never in `SUPPORTED_EXTENSIONS` in the first place —
the exact bug behind LibreChat's own "the agent said it made a file, no
download appeared" symptom that motivated building preview-server at all,
identified back then and deprioritized in favor of preview-server's
tool-call path. This is that bug, finally fixed, and wired to
preview-server instead of (only) LibreChat's own limited file-attachment
UI.

**How it works, end to end**: `uploadOneFile()` already streams a
qualifying output file to `file_server` over plain HTTP. After that
existing upload succeeds, `forwardToPreviewServer()` reads the file
(`.fasta`/`.fa` only — small text, no need for the primary upload's
streaming discipline, which exists for the general large/binary case) and
`POST`s it to `PREVIEW_SERVER_URL/ingest/{kind}`
(`lab-agent-preview-server`'s new plain-HTTP ingest route, not an MCP
tool — the agent never calls it directly). Any failure here (network,
non-2xx, `PREVIEW_SERVER_URL` unset) is caught and logged, never fails
the job or the primary upload.

`docker-compose.local-dev.yml`: `sandbox`'s environment gets
`PREVIEW_SERVER_URL=http://host.docker.internal:8012` (preview-server
runs directly on this host via systemd, same address
`lab-agent/librechat.yaml`'s own preview-server MCP entry already uses)
— empty/unset makes the whole forward step a no-op, so any other
deployment of this fork is unaffected unless it opts in. Also added
`extra_hosts: host.docker.internal:host-gateway` to `sandbox` (needed for
the *container* to resolve that address at all, before `entrypoint.sh`
ever gets to pre-resolve it for the guest — confirmed missing here;
`lab-agent/docker-compose.yml` already needed the same entry for the same
reason).

**Not done**: the agent still can't discover the resulting preview URL
in the same turn as a return value — no `previewUrl` field was added to
the upload response, deliberately, to keep the `job.ts` diff smaller. It
calls `list_previews()` (an existing, unrelated MCP tool on
preview-server) to find the link afterward instead. Revisit if that
extra round trip proves annoying in practice.

**Test coverage gap, noted rather than silently skipped**: neither
`SUPPORTED_EXTENSIONS`/`isSupportedOutputFilename` nor `uploadOneFile`
have any existing unit tests in this codebase to extend — confirmed by
grepping the whole test suite. Building a from-scratch `Job`-construction
test harness for this one change felt like a larger, more invasive
addition than the change itself, working against the "small diff"
motivation for this whole file. Verified instead by `npx tsc --noEmit`
(clean — introduces no new type errors, checked against the pre-existing,
unrelated errors already present in a few `.test.ts` files) and a real
live end-to-end run against the actual rebuilt `sandbox` container: a
job posted to `/api/v2/execute` wrote a real 52-record FASTA file via
Python code execution, and the resulting preview
(confirmed via `lab-agent-preview-server`'s own stored JSON, its `/view`
page, and `/download`) had all 52 records, byte-for-byte.

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
