Issue:    ai-gladiators/ai-gladiators-competitions#32
Title:    Build the crossbar-v1 two-node target (source-of-truth: configs, workload, referee, manifest)
URL:      https://github.com/ai-gladiators/ai-gladiators-competitions/issues/32
State:    OPEN at freeze
Author:   @oberstet
Updated:  2026-11-20T19:43:33Z
Comments: 0
Frozen:   2026-11-20T19:44:31Z by `just new-branch`
SHA-256:  5b77ddd3dfd1485ac1a5ecff929d90433d93342aeb0dd75a0cc3dac8b5e849cf (of everything below the line)

---


## Why

`crossbar-v1` is defined (the neutral competition text already names a two-node
Crossbar.io mesh: two routers **A**/**B**, one router-to-router link forwarding the
application realms, the nine-client workload distributed across both nodes, Setup-2
TLS with `channel_binding` not configured, SUT pinned at crossbar `163dc854`). What
does not exist yet is the **target itself** — the two router configurations, the
link configuration, the distributed workload, the referee, and the instantiate
manifest.

Per the target-placement design (competitions#30), the target lives **source-of-truth
in competitions** and is materialized into `ai-gladiators-briefing` before a match.
This issue builds the v1 target here. A follow-up (briefing-side) instantiates it.

**Boundary (non-negotiable):** competitions is public and **denied from `red`'s
corpus**, and briefing (where this lands) **is** `red`'s corpus. Everything here is
the *target a contestant faces* — published per Kerckhoffs. **No exploit tradecraft**
(no description of how the mesh is broken, no relay/oracle/key-reuse/channel-binding
bypass mechanics) belongs in any file this issue touches; that stays in
`ai-gladiators-operator`. The configs state plain target facts (e.g. "`channel_binding`
is not configured on the link") and nothing about exploiting them.

## What to build

Under `targets/crossbar/two-node/`, mirroring the `single-node/` layout:

### 1. Two router configurations

- `nodeA/.crossbar/config.json`, `nodeB/.crossbar/config.json` — each serves the
  same three realms as v0 (`public` anonymous; `application1`, `application2`
  WAMP-Cryptosign static auth), with the same per-realm roles/permissions as the
  single-node config.
- Client-facing transports run over **TLS** (v0 is plaintext TCP/WS; v1 adds TLS).

### 2. The router-to-router link (the thing v1 adds)

- One **`rlink`** stanza joining the two routers so `application1` + `application2`
  are shared across the mesh (an event published on an app realm at one node reaches
  that realm's members at the other). Its direction, the realms it carries, and its
  transport are all **published in the config** (Kerckhoffs).
- The link transport is **TLS**; the link authenticates with **WAMP-Cryptosign**;
  **`channel_binding` is not configured** on the link (Setup-2). State these as
  config facts only.

### 3. Distributed workload

- Reuse v0's nine-client `workload.py`; add a **client→node assignment** so each app
  realm has members on **both** routers (so a realm's peers may sit on either node).
  Keep the tick cadence and the SHA-256-digest-per-tick flag scheme unchanged.

### 4. Referee (per-target, mirroring v0)

- `targets/crossbar/two-node/referee/verify_flags.py`, `verify_events.py` — a
  per-target copy (v0 already keeps its referee under `single-node/referee/`). Adapt
  only as needed to ingest the nine clients' logs when they are **split across two
  nodes** (same verdicts: `counts` / `injected`, same S1/S2/S4/S5 scoring). If no code
  change is needed, copy verbatim and record that.

### 5. The competition text, sourced here

- Move `crossbar-v1.md` to **source-of-truth in competitions** (it was hand-written
  directly into briefing in briefing#1). Fix its referee links to the per-target home
  `../targets/crossbar/two-node/referee/...`. briefing's copy becomes a materialized,
  provenance-stamped instantiate output (the follow-up issue retires briefing's
  top-level `referee/`).

### 6. Smoke harness

- `smoke.sh` + `smoke_inject.py` analogous to v0: bring up **both** routers and the
  distributed nine clients for a bounded window; a parameterized recipe in `justfile`
  (mirror `smoke`).

### 7. VERIFY-at-build (target correctness, not tradecraft)

A build-time check that the target is the **mesh it claims to be**: the rlink's
authrole has **publish + subscribe** permission on each **forwarded** app realm
(`application1`, `application2`) on **both** nodes, so cross-node delivery of those
realms actually happens. This verifies a published configuration property; it is not
an exploit. Wire it into `just check` (or a dedicated recipe) so a misconfigured
target fails the build rather than a match.

### 8. The instantiate manifest

- `competitions/crossbar-v1.instantiate.toml` — the **explicit allowlist** of
  contestant-facing files copied into briefing (both node configs, the link config if
  separate, the workload + client project, the two-node README, `web/`, the referee,
  and `crossbar-v1.md`). Nothing from `matches/`, operator notes, or reviews is
  eligible. This is the disclosure boundary for v1, same mechanism as
  `crossbar-v0.instantiate.toml`.

## Secrets (never in git)

The only secrets are the **flag values** and the **private keys** — the clients'
Cryptosign private keys and the routers' **node** and **TLS** private keys. Ship a
setup step/recipe that generates keys + certs locally at match setup; only **public**
keys and pinned authids go in the committed config. No private key material, no flag
value, ever enters the repo.

## Deliverables

1. `targets/crossbar/two-node/` — two node configs + rlink, TLS transports,
   distributed workload, per-target referee, README, smoke harness.
2. VERIFY-at-build check wired into a recipe.
3. `crossbar-v1.md` as source-of-truth here, referee links corrected.
4. `competitions/crossbar-v1.instantiate.toml` (the allowlist).
5. Key/cert generation step documented; nothing secret committed.

## Verify

- `just check` passes, including the VERIFY-at-build rlink-permission assertion.
- `just smoke-two-node` (or equivalent) brings up both routers + nine distributed
  clients for the window with no errors.
- A dry `just instantiate crossbar-v1 ../ai-gladiators-briefing` from a clean
  briefing checkout stages exactly the manifest's files (the follow-up issue lands it).

## Sequencing

Precedes the briefing-side **instantiate crossbar-v1** issue (the mirror of
briefing#3 for v0), which copies this target into `red`'s corpus and retires
briefing's top-level `referee/`.

---

## Comments at freeze (0)

