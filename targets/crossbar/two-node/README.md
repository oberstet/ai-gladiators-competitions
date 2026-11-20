# Target: Crossbar.io, two-node mesh

Two [Crossbar.io](https://crossbar.io) routers (**node A** and **node B**), joined
by one router-to-router link (`rlink`), and nine
[AutobahnPython](https://github.com/crossbario/autobahn-python) clients running a
seeded workload spread across both routers. Both router configurations, the link
configuration and the client code are published here in full. Only the keys stay
out of the repository.

This target extends [`crossbar-v0`](../single-node/) (one router): the client code,
the referee and the flag scheme are the same; what is new is the **second router and
the link between them**. Read the single-node README first for the client, the flag
scheme and the referee; this README covers only what the mesh adds.

## The system under test

Everything in the single-node target's call path is here too (the WAMP stack, the
interpreter, the TLS library, the event loop). The mesh adds one more element in the
path of a cross-node call: **the router-to-router link**.

| layer | routers | the link | clients |
|---|---|---|---|
| machine | one KVM/QEMU VM per node (two), Debian 13 | — | one VM per client (nine), Debian 13 |
| interpreter | CPython | CPython | CPython |
| software | Crossbar.io on Twisted, epoll reactor | an rlink session on each router | AutobahnPython on asyncio |
| transport | WebSocket/RawSocket over TLS | **RawSocket over TLS** | WebSocket/RawSocket over TLS |
| serializer | CBOR | CBOR | CBOR |
| authentication | anonymous (`public`), WAMP-Cryptosign (`application1`, `application2`) | **WAMP-Cryptosign (node keys)** | the same |

The System Under Test is **Crossbar.io at commit `163dc854`** with the Autobahn
libraries it pins, the same pin as `crossbar-v0`.

## The two routers

Each node runs the same three realms as the single-node target:

| realm | authentication | role(s) | meshed across the link? |
|---|---|---|---|
| `public` | anonymous | `guest` | no |
| `application1` | WAMP-Cryptosign | `device`, `rlink` | yes |
| `application2` | WAMP-Cryptosign | `device`, `rlink` | yes |

[`nodeA/.crossbar/config.json`](nodeA/.crossbar/config.json) and
[`nodeB/.crossbar/config.json`](nodeB/.crossbar/config.json) each define, besides the
client-facing `universal` transport over TLS (as in v0):

- an **rlink listener** — a RawSocket-over-TLS transport with a WAMP-Cryptosign
  `cryptosign` principal named `rlink`, whose `authorized_keys` is the **other
  node's public key**. The principal pins no realm, so the peer's link is admitted
  into whichever application realm it requests, with role `rlink` there;
- on each application realm, an **`rlink` entry** that dials the other node's rlink
  listener and forwards that realm. Each node forwards its own local traffic to the
  peer (`forward_local_events` / `forward_local_invocations` true) and does not
  re-forward what it receives (`forward_remote_*` false), so the two links form a
  full mesh without looping an event back.

The effect: a **call** to a procedure registered on `application1` at node A is routed
to the callee at node B, and the reverse; likewise for `application2`. `public` is not
meshed and is local to each node.

> **What forwards across the rlink in this SUT pin.** This Crossbar.io commit forwards
> **exact** registrations and subscriptions across an rlink (so RPC to exact-registered
> procedures routes across the mesh), but does **not** forward **prefix/wildcard
> subscriptions**. The workload registers its `echo` procedures with exact match, so
> cross-node calls work; its tick subscription is a prefix subscription, so ticks are
> **not reliably forwarded across the mesh** (some may cross once an exact subscription
> on the tick topic happens to bridge the nodes, but this is not guaranteed). This
> target's shared-realm property is therefore demonstrated and scored through **calls**
> (and through the link's own Cryptosign endpoint), not through cross-node events. (A
> note on the subscription-forwarding behaviour has been raised upstream.)

### The link's transport (this competition's setup)

Stated as plain target facts (the authoritative values are in the two configs):

- the client-facing transports and the rlink both run over **TLS**;
- the rlink authenticates with **WAMP-Cryptosign**, each router signing with its
  **node key**;
- **`channel_binding` is not configured** on the link: the router does not require
  the peer to bind its authentication to the TLS channel.

### A cross-node caller's disclosed identity

A plain behaviour of the SUT's mesh, worth stating because the workload records it:
when a call is **forwarded across the rlink**, the callee sees the forwarded
caller's **authrole** (`device`) disclosed in place of its **authid**. Same-node
calls disclose the caller's authid as usual. The workload notes this on cross-node
calls; it is a property of the link, not a match violation, and the referee scores
from events, not from this call-side note.

### Keys

There is no key material in this repository.

- **The TLS key and certificate** of each node (`<node>/.crossbar/server.key`,
  `server.crt`) are generated per deployment. Each node must also **trust the other
  node's certificate** for the rlink: the rlink transport names
  `ca_certificates: ["peer.crt"]`, so a deployment places the peer's certificate at
  `<node>/.crossbar/peer.crt`. For a local run, make a self-signed `localhost` pair
  per node (as in the v0 README) and copy each node's `server.crt` to the other's
  `peer.crt`.
- **The node keys** (`<node>/.crossbar/key.priv`, `key.pub`) are generated by
  Crossbar.io on first start (`crossbar keys --cbdir <node>/.crossbar`). Each node's
  **public** key goes into the other node's rlink-listener `authorized_keys`. A
  public key is not key material; the private node key never leaves its node. The
  committed configs carry example node public keys as placeholders.
- **The Cryptosign principals** `client04`–`client09` start empty, as in v0; each
  client's public key is added to its principal in **both** node configs.

## The workload across the mesh

The client is [`client/workload.py`](client/workload.py), unchanged from v0. What
differs is only **which node each client connects to**, chosen at deployment so that
every application realm has members on **both** routers — so a realm member reaches
a realm member on the other node only through the rlink. The local smoke test uses:

| node | clients |
|---|---|
| A | `client01` `client02` `client03` (public), `client04` `client05` (application1), `client07` (application2) |
| B | `client06` (application1), `client08` `client09` (application2) |

`public` is local, so its three members sit together on node A. The application
realms span the mesh, so a member's **peer-calls** to a member on the other node are
routed across the rlink (ticks, being prefix-subscription events, are not reliably
forwarded across the mesh in this SUT — see the box above).

## VERIFY-at-build: the target is the mesh it claims to be

[`verify_target.py`](verify_target.py) reads both node configs and refuses unless the
target actually has the property a contestant is told to attack: two routers, each
application realm carrying an `rlink` role with call and register (and publish/subscribe), an rlinks
entry forwarding each application realm to the peer over TLS, and a TLS rlink
listener that authorizes a peer key. It reads only the published configuration and is
a target-correctness check, not an exploit. Standard library only.

```console
python3 targets/crossbar/two-node/verify_target.py
```

A misconfigured target (a missing rlink, an rlink role without publish or subscribe,
a listener that authorizes no peer, an rlink role missing call or register) fails the build rather than a match.

## Smoke test

[`smoke.sh`](smoke.sh) runs the whole two-node target on one machine:

1. it copies both node directories to a temporary directory;
2. it generates a throwaway TLS pair per node (cross-trusted as each other's
   `peer.crt`), a node key per node, and throwaway client keys, and writes each
   node's public key into the other's rlink-listener `authorized_keys`, the client
   keys into both nodes, and the four ports into both configs;
3. it starts both routers on the epoll reactor and waits for the web page of each
   over TLS and for the two router-to-router links to come up;
4. it starts the nine clients split across the two nodes (every application realm on
   both), so a realm member's calls to a member on the other node are forwarded over
   the rlink — the **cross-node RPC** that proves the mesh;
5. it requires the routers to refuse an unknown Cryptosign key and an anonymous
   client in `application1`, each backing off after every refusal;
6. [`smoke_inject.py`](smoke_inject.py) injects three events on `application1` at
   node A that no member published as such (replaying/forging a node-A member), so
   they reach the node-A `application1` members;
7. it requires every client to publish well-formed ticks, every **cross-node call**
   to be forwarded and answered with the right payload and callee (only the disclosed
   caller identity differs — the authrole note above — which is not a match
   violation), no flag to appear outside its publisher's record, and the injected
   events to be seen at the node-A members;
8. it runs `verify_flags.py` on five claims (counts, counts, control, obtained,
   invalid);
9. it runs `verify_events.py` and requires exactly three injected events at each
   **node-A** member of `application1` (`client04`, `client05`), and none elsewhere.

From the repository root:

```console
just smoke-two-node ~/work/autobahn-crossbar/crossbar/.venvs/cpy314/bin/crossbar
```

`smoke.sh` itself takes `PYTHON` (the python of the client's venv), `CROSSBAR`, and
the four ports `PORT_A_CLIENT` / `PORT_A_RLINK` / `PORT_B_CLIENT` / `PORT_B_RLINK`
(defaults 8080/8081/8082/8083) from the environment, and refuses to start when a
port is taken. To keep it off the host's network, run it in its own network
namespace (`unshare -rn --pid --fork --mount-proc …`).

---

Code and configuration in this directory are MIT ([LICENSE](../../../LICENSE)); this
README is CC BY 4.0 ([LICENSE-docs](../../../LICENSE-docs)).
