# Gryth Cooperation Demo — Proposal

Status: **draft** — decision-ready; review and prune before execution.

Companion to `GrythVision.md`. Scope: **demo code only**, not production. This
document specifies a demo that pulls the existing standalone glade share-first
demo (`../../glial-dev/glade/demo`) together with the gryth-ui React desktop to
show **two gryth-ui desktops cooperating** when their users hold different but
*intersecting* sets of workspaces.

All paths are absolute or repo-relative. Every API cited as REAL was confirmed
by reading the source; anything not yet built is flagged **STUB**, **NEW**, or
listed as an open question. Rule while executing: if a seam isn't there, build
the *named* seam — don't fake the call.

---

## 1. Purpose — the one thing to prove

> **Two gryth-ui desktops, run by two different users, hold different
> *intersecting* sets of workspaces. Changes in a *shared* workspace converge
> across both desktops in real time; changes in a *private* workspace stay with
> their owner — with no per-surface special-casing in UI code.**

Concretely: User A holds `{W1, W2}`, User B holds `{W2, W3}`. Only **W2** is
shared. A's edit in W2 appears on B's desktop; A's edit in W1 never reaches B,
and B never even learns W1 exists. The intersection is **not** enforced by
application logic or an ACL tree — it falls out of the glade addressing model
(`share` × `key`) and the per-user grant.

Secondary (already true in glade, re-shown in gryth-ui): two browser tabs run by
the **same** user converge.

Non-goals: real auth, encryption, p2p mesh, live PTY output streaming,
client-side disk persistence, conflict UI. All deferred behind existing seams
(§7).

---

## 2. Current state of each piece

Six pieces already exist. The demo wires them together; it builds **one** new
package (a gryth-ui glade-binding seam) plus **one** schema change (a minimal
per-workspace doc scope) and adds **one** genuinely new doc-scope surface for the
first slice.

### 2.1 Glade substrate node (Rust) — REAL, multi-share by construction

- Node loop, routing, store, sessions: `../../glial-dev/glade/node/src/{server,router,store,chain,session}.rs`.
- One node hosts **arbitrarily many shares** at once; a `share` is an opaque
  string (`"doc:W2"`, `"account:alice"`). The node has no app semantics.
- The atomic addressable unit on the wire is the triple **`(share, glade_id,
  key)`** — a "zone-surface". Ops are keyed by `(share, glade_id, key, origin)`
  with a per-origin monotonic `seq` + `prev`-hash chain.
- Routing is by **exact triple**: `Router::route(from, share, glade_id, key) ->
  Vec<SessionId>` (`router.rs:48`) fans an op only to sessions subscribed to that
  exact triple, minus the originator. A private-keyed op (`key="self:alice"`)
  cannot reach a commons-only subscriber, and an op in `doc:W1` cannot reach a
  session that never subscribed to `doc:W1`. This is **privacy-by-keying and
  by-subscription, not by permission check.**
- One session may register interest in **many** triples (`server.rs:120`
  `Subscribe` registers `(sid, share, glade_id, key)`; nothing limits a sid to
  one share). So *one socket can carry many workspaces* — relevant to §4.4.
- Identity/capability fields exist on the wire (`Hello.principal`,
  `Hello.capability`) but are **unenforced** in the current milestone (M-LIMP) —
  the node serves any subscribe/append. Acceptable, and honest, for a localhost
  demo.

**What is proven vs not:**
- **Proven end-to-end (real node):** commons-converges / private-isolates *within
  one share*, by `key`. `../../glial-dev/glade/grip-share/test/node_integration.test.ts`
  ("zones" test) runs alice + bob through the real node over a single
  `share:"doc:1"`, varying only the `key`.
- **Structurally guaranteed but NOT yet exercised end-to-end:** cross-*share*
  isolation — the disjoint-set property (A holds `doc:W1`+`doc:W2`, B holds
  `doc:W2`+`doc:W3`, and `doc:W1` ops never reach B). It follows from
  `route(...)` keying on the full triple (covered by `router.rs` unit tests), but
  **no test subscribes one client to two distinct `doc:*` shares.** This is
  exactly what Slice 2 first exercises (§5, §6 item 13).

### 2.2 client-ts — REAL transport + fold

- `Session` (`../../glial-dev/glade/client-ts/src/session.ts`): `append(share,
  gladeId, shape, payload, key)`, `applyRemote(ops)`, `fold(share, gladeId,
  shape, key)`, `dump()`. **`share` is a per-op parameter** — one `Session`
  holds ops for many shares; the store partitions by `(share, gladeId, key,
  origin)`.
- `GladeClient` (`.../client.ts`): pure WebSocket transport — `connect(url)`,
  `subscribe(share, gladeId, key?)` (share per call), `sendOps(ops)`, `onOps`
  hook. No app logic. One client = **one socket**.
- Folds (`.../fold.ts`): `foldValue` = LWW by `(lamport, origin)`; `foldLog` =
  ordered by `(lamport, origin, seq)`. Deterministic, cross-language pinned
  (`taut/corpus/glade.*`).
- Store (`.../store.ts`): in-memory. IndexedDB deferred (same interface). For the
  demo, node-disk persistence + in-memory client is enough.

### 2.3 grip-share binder — REAL, generic, transport-agnostic

- `GripShareBinder` (`../../glial-dev/glade/grip-share/src/binder.ts`).
  Constructor `(grok, session, codecs?, scope)`. Walks `grok.listSharedTaps()`,
  resolves each tap's `ShareDecl` to an `Addr {share, key}` via the `Scope`,
  hydrates from `session.fold()`, subscribes to local tap changes → appends ops,
  applies remote ops back to taps with an `applying` echo guard.
- Public API the demo uses: `bind()`, `subscriptions()` (returns `[{share,
  gladeId, key}, …]`), `appendLog(gladeId, entry)`, `applyRemote(ops)`,
  `resync()`, `dispose()`, and the `onLocalOps` hook.
- Carries **no grip-core imports** — it operates purely on the structural share
  hooks (`getShareValue`/`applyShareValue`/`subscribeShare`/`share`). The exact
  grip-core it targets is the **same symlinked package** gryth-ui uses, so
  "binder reusable unchanged" holds — *with one real constraint*, §4.4.
- **Per-shape behaviour matters (binder.ts):** for `shape !== "log"`, `bind()`
  subscribes to the tap and mirrors the whole value on every local change. For
  `shape === "log"`, `bind()` **skips** `subscribeShare` — a log surface emits
  ops **only** via `binder.appendLog(gladeId, entry)`. A normal `.set()/.update()`
  on a log-backed atom produces **no op**. Any log surface needs an explicit
  append call site (the demo's `postActivity` pattern in `demo/src/glade.ts`).

### 2.4 grip-core share feature + grip-react — REAL, no changes needed

- `ShareDecl` and the tap share hooks live in
  `../../glial-dev/grip-core/src/core/tap.ts`; `AtomValueTap`
  (`.../atom_tap.ts`) implements all three hooks and its factory accepts
  `share?: ShareDecl` in opts (`createAtomValueTap(grip, { initial, handleGrip,
  share })`, `atom_tap.ts:81,191`). `applyShareValue` routes through `set`, so a
  remote apply round-trips into every `useGrip` reader.
- `Grok.listSharedTaps()` collects every share-declared tap.
- `useGrip` is share-unaware and needs **no change**: a shared value reads
  identically to a local one.

### 2.5 Manifest / Grant / Scope model — REAL declaration layer

- `../../glial-dev/glade/grip-share/src/manifest.ts`: `Manifest`, `Grant`,
  `manifestScope(manifest, grant)`, `surfaceDecl(manifest, gladeId)`,
  `manifestCodecs(manifest, byType)`.
- Working example + stub identity: `../../glial-dev/glade/demo/src/manifest.ts` —
  `WORKSPACE_MANIFEST` (surfaces `app:notes`/`app:activity`/`app:selection`/`app:status`)
  and `stubGrant(user, doc)` (note: **single** `doc`, emits one `doc:{doc}` allow
  entry).
- Wiring reference to port: `../../glial-dev/glade/demo/src/glade.ts` — builds
  `Session`, `manifestScope`, `manifestCodecs`, `GripShareBinder`, `GladeClient`,
  wires `client.onOps → binder.applyRemote` and `binder.onLocalOps →
  client.sendOps`, connects, then for each `binder.subscriptions()` calls
  `client.subscribe(share, gladeId, key)`. **`manifestScope` substitutes a single
  `{doc}` from `grant.session.doc`** — one scope addresses one workspace (§4.4).

### 2.6 gryth-ui desktop + plugins — REAL shell, mock-backed surfaces

- Composition root: `gryth-ui/src/bootstrap.tsx` (wraps `<GripProvider
  grok={grok} context={main}>`); tap registration entry `gryth-ui/src/taps.ts`
  (`registerAllTaps()`).
- Shared singletons: `gryth-ui/packages/plugin-api/src/runtime.ts` (`registry`,
  `grok`, `main`).
- Contract grips: `gryth-ui/packages/plugin-api/src/grips.ts` — `WORKSPACE_NAME =
  defineGrip('Doc.WorkspaceName', '')` (default `''`; the initial value
  `'mock-workspace'` is set by `WorkspaceNameTap` in `src/taps.ts`, **not** in
  grips.ts).

**Scope of each plugin grip (this distinction is load-bearing — corrects an
error in the first draft):**

| Plugin | Grip | Registration | Scope | Shape |
|---|---|---|---|---|
| (root) | `WORKSPACE_NAME` | `src/taps.ts` `registerTap` | **doc (root)** | value |
| workspace | `WORKSPACE_LIST` | `workspace/src/index.ts:31` `registerTap` | **doc (root)** | value (record array) |
| terminals | `TERMINAL_SESSIONS` | `terminals/src/index.ts:31` `registerTap` | **doc (root)** | value (record array) |
| vm | `VM_MACHINES`/`VM_BASES`/`VM_PROVIDERS`/`VM_PROFILES` | `vm/src/index.ts:38-41` `registerTap` | **doc (root)** | value |
| chat | `CHAT_TRANSCRIPT`, `CHAT_DRAFT` | `chat/src/index.ts` **`tabTaps`** | **instance (per-tab)** | array atom |
| code | `WTA`, viewer state | `code/src/index.ts:34` **`tabTaps`** | **instance (per-tab)** | value |
| workspace | viewer/graph state | `workspace/src/index.ts:24` **`tabTaps`** | **instance (per-tab)** | value |

Only the four **root**-registered grips are doc-scope-ready. `CHAT_TRANSCRIPT` is
explicitly per-tab — `chat/src/index.ts` comments *"No root tap: this tool's
state is entirely per-tab ... every chat window is an independent conversation"*.
A share decl on a `tabTaps` grip would bind a per-tab atom to a shared surface,
which is wrong. **Any per-tab surface must first be lifted into a root or
per-workspace context before it can carry a share.**

- **No-React-state rule enforced** (`gryth-ui/dev-docs/CodingRules.md`,
  `scripts/no-react-state.test.mjs`): `useState/useEffect/useRef/useReducer/
  useMemo/useCallback/useLayoutEffect` are banned across all package `src` dirs;
  all state lives in grips + atom taps. This is exactly what makes the
  mock→glade swap transparent to components — and a hard gate the demo must pass.

### 2.7 The gaps (what doesn't exist yet)

1. **No glade binding in gryth-ui.** The binder seam (§2.3) has never been
   attached inside gryth-ui; surfaces are mock atoms.
2. **No workspace scope.** `WORKSPACE_NAME` is a single string; the sidebar shows
   it **read-only** (`Desktop.tsx:703` `<div className="sidebar-workspace">`).
   There is no workspace selector, no per-workspace doc grips, no notion of a user
   *holding a set* of workspaces.
3. **No identity/presence surface.** `PrincipalRef` exists only as a type
   (`plugin-api/.../registry.ts`); nothing renders "I am alice" or "who else is
   here".
4. **The binder maps each tap to exactly one address** (§4.4) — so one binder
   instance serves one workspace.

---

## 3. Core design insight — intersecting sets fall out of `share` × `key`

The whole demo rests on one mapping:

> **A gryth "workspace" is a glade `share`.** `W2` ⇒ `share = "doc:W2"`.

Within a workspace, glade's `key` (zone) partitions data:

- **commons** zone, `key = ""` — everyone in that share converges (LWW for
  values, ordered interleave for logs).
- **private** zone, `key = "self:<user>"` — keyed to one principal; routing
  filters it so only that user sees it, even inside a shared workspace.

A user's **set of workspaces** is exactly the set of `doc:*` `share` values their
`Grant.allow` enumerates. Intersection is **set intersection over those `share`
strings**. No tree, no ACL graph.

### 3.1 The A={W1,W2} / B={W2,W3} walk-through

Manifest (one, static; shape mirrors `demo/src/manifest.ts`; surface ids are
gryth's own — the demo's `app:*` ids are a naming reference, not reused
verbatim):

```
params:   { self: {from:"identity"}, doc: {from:"session"} }
domains:  { doc: {share:"doc:{doc}"}, account: {share:"account:{self}"} }
zones:    { commons: {key:""}, private: {key:"self:{self}"} }
surfaces:
  "gryth:notes":     { domain:"doc", zone:"commons", shape:"value", type:"Text" }
  "gryth:wsmeta":    { domain:"doc", zone:"commons", shape:"value", type:"Json" }
  "gryth:selection": { domain:"doc", zone:"private", shape:"value", type:"Text" }
  "gryth:chat":      { domain:"doc", zone:"commons", shape:"log",   type:"ChatLine" }  # later slice
```

A's grant (identity authenticated tomorrow; **STUB** from URL today;
`stubGrant` extended to take a *list* of docs — small NEW work, §6 item 4):

```
identity: { self: "alice" }
session:  { doc: "W2" }                 // the workspace currently focused
allow: [
  { share:"doc:W1", keys:["", "self:alice"] },
  { share:"doc:W2", keys:["", "self:alice"] },
  { share:"account:alice", keys:[""] }
]
```

B's grant:

```
identity: { self: "bob" }
session:  { doc: "W2" }
allow: [
  { share:"doc:W2", keys:["", "self:bob"] },
  { share:"doc:W3", keys:["", "self:bob"] },
  { share:"account:bob", keys:[""] }
]
```

Resolution and outcome, per surface:

| Surface (zone) | A in W2 → addr | B in W2 → addr | Result |
|---|---|---|---|
| `gryth:notes` (commons) | `doc:W2, key=""` | `doc:W2, key=""` | **converges** A↔B |
| `gryth:wsmeta` (commons) | `doc:W2, key=""` | `doc:W2, key=""` | **converges** A↔B |
| `gryth:selection` (private) | `doc:W2, key="self:alice"` | `doc:W2, key="self:bob"` | **isolated** (different chains) |
| `gryth:notes` in W1 | `doc:W1, key=""` | — (not in B's allow) | A only; B never subscribes |
| `gryth:notes` in W3 | — (not in A's allow) | `doc:W3, key=""` | B only; A never subscribes |

What converges: **W2 commons** (notes, workspace metadata). What stays private:
**W2 private selection** (alice's vs bob's — same share, different key → different
hash chains → routing never crosses them). What never crosses at all: **W1**
(A-only share) and **W3** (B-only share) — B's binder emits no `doc:W1`
subscription, so the node never routes `doc:W1` ops to B.

Trust property: `key="self:{self}"` is filled from `grant.identity`, which (in
the real model) is agent-authenticated and non-forgeable. In the demo `self` is
client-settable (URL param) — fenced in §5/§7.

---

## 4. Architecture — hosting share-first surfaces inside gryth-ui

The binder attaches at the gryth-ui composition root and binds **doc-scope root
grips** the plugins already declare (or the one new grip Slice 0 adds).
Components don't change.

### 4.1 Where the binder attaches

In `gryth-ui/src/bootstrap.tsx`, after `registerAllTaps()` (so share-declared
taps are present in the grok), add a `startGladeSync()` step mirroring
`demo/src/glade.ts`:

```
session = new Session(schema, origin)
scope   = manifestScope(GRYTH_MANIFEST, grant)
codecs  = manifestCodecs(GRYTH_MANIFEST, CODECS_BY_TYPE)
binder  = new GripShareBinder({ listSharedTaps: () => grok.listSharedTaps() },
                              session, codecs, scope)
client  = new GladeClient(schema, origin, session)
client.onOps      = (ops) => binder.applyRemote(ops)
binder.onLocalOps = (ops) => client.sendOps(ops)
await client.connect(NODE_URL)
binder.bind()
for (const s of binder.subscriptions()) await client.subscribe(s.share, s.gladeId, s.key)
```

Place this in a small new package `@grythjs/glade-sync` (peer to `plugin-api`),
so plugins stay glade-unaware. Bootstrap calls `startGladeSync(grok, grant)`.

### 4.2 How the seam swaps mock for glade with no consumer rewrite

The **root** plugin grips are already `AtomValueTap`s with the share hooks. Give
each shareable **root** grip a `share: surfaceDecl(GRYTH_MANIFEST, gladeId)` on
its existing `createAtomValueTap`. `binder.bind()` then hydrates the atom from
folded glade state and forwards its `.set()/.update()` as ops (value surfaces) or
expects `appendLog` (log surfaces, §2.3). The mock initial value is the local
pre-connect value; on connect it converges. **No component changes; no matcher
infra.** This is the actual mechanism for this demo — exactly what
`demo/src/taps.ts` does (`createAtomValueTap(GRIP, { initial, handleGrip, share:
surfaceDecl(...) })`).

**Out of scope — do not build:** the `withOneOf()` provider-matcher described in
`gryth-ui/dev-docs/PluginMigration.md` selects between two *different backing
taps* (mock vs a `pty`/`gryth` provider) behind one grip. That is an orthogonal
future concern for mock-vs-real-backend selection; it does **not** replace a tap
with the binder (the binder attaches share hooks to the *same* atom). The demo
needs none of it.

### 4.3 Where per-workspace manifest/grant binding lives

The manifest is **static** and shared by all workspaces (templates, not per-W
entries). Per-workspace-ness lives in:

1. **The grant's `allow` list** — enumerates which `doc:*` shares this user holds.
2. **Scope resolution** — `scope.resolve(decl)` fills `{doc}` from grant session.
   Addressing *multiple* workspaces requires varying `{doc}` per binding (§4.4).

The grant is minted in `@grythjs/glade-sync` (STUB: from URL; real: from an agent
endpoint). It is the single app↔authorization boundary — demo code does not
change when real auth lands, only the line that produces `grant`.

### 4.4 The one real new constraint — multi-workspace from one user

`Session`, `Store`, `GladeClient`, and the node **already span many shares** (§2.1,
§2.2). The genuine gap is in the **binder**: `binder.bind()` resolves each tap to
exactly **one** `Addr` (`addrs: gladeId → one Addr`), so one binder serves one
workspace. Two honest implementations:

- **Option M1 — one stack per workspace (recommended for first cut).** For each
  `doc:*` in `grant.allow`, instantiate an independent `{Session, GripShareBinder,
  GladeClient, scope}` bound to a grok **sub-context** whose taps carry that
  workspace's `{doc}` (§4.5). Pros: uses only proven APIs; the binder/session/node
  are untouched. Cons: N sockets for N held workspaces. Fine on localhost.
- **Option M2 — one socket, scope varies `{doc}` per surface.** Refactor the
  binder so a shareable tap fans to multiple `(share,key)` and ops are routed to
  the right binding by share. Cleaner long-term (single socket, leveraging the
  node's many-triples-per-session support) but a real binder change. **Defer
  past the demo.**

**Recommendation:** M1. It touches only the new `glade-sync` package and the
per-workspace context, leaving the proven substrate alone.

### 4.5 Workspace scope (the schema change) — minimal, fenced

Slice 2 needs a user to hold a *set* of workspaces with per-workspace doc state.
Minimal version:

- A `WORKSPACE_ID` grip (the active workspace) + a **writable** sidebar selector
  that sets it (replaces the read-only `Desktop.tsx:703` label).
- A per-workspace grok **sub-context** keyed `ws:{id}`, modeled on the existing
  per-tab `tab:{id}` contexts (`gryth-ui/packages/desktop/src/tabContexts.ts`).
  The doc-scope root taps (notes/wsmeta/selection) are registered **within** each
  workspace context so each workspace has its own atom; environ/instance grips
  (window layout, drag, hover, viewer state) stay where they are.

**Fence (explicitly NOT in the demo):** promotion semantics, mirrored-vs-
independent desktops, shared-focus defaults — all open in `GrythVision.md`.

**Cheaper fallback if §4.5 proves heavy:** pre-seed exactly two `ws:{id}`
contexts per user from config (no live reparenting, no general selector beyond a
two-item toggle). This still proves the headline with far less infra; choose it
if the general per-workspace-context machinery threatens the timeline.

### 4.6 Legibility — REQUIRED for Slice 2, not optional

Two converging tabs look identical to one tab unless the viewer can *see* the
structure. Slice 2 acceptance requires, on each desktop:

- **identity badge** — "I am alice" (a glade-`account` or local grip).
- **writable workspace selector** — shows the user's held set; switching it is
  how W1-vs-W3 absence becomes observable.
- **shared-vs-private affordance** — a glade-backed `gryth:wsmeta`/presence
  surface so "W2 is shared (bob is here), W1 is yours" is visible, not inferred.

### 4.7 No-React-state honored

The binder lives in plain TS (`glade-sync`). Connection status, active workspace,
identity, and all shared values are grips read via `useGrip`; mutations go
through tap handles. Mandatory; enforced by `scripts/no-react-state.test.mjs`.

---

## 5. Smallest honest first slice + staged path

### Slice 0 — one shared surface, one shared workspace (the spine)

**Goal:** prove convergence end-to-end through gryth-ui with the *least* new code,
on a surface that is *genuinely* doc-scope.

- **Surface:** a **NEW** root-registered doc-scope value grip `Doc.Notes`
  (commons, LWW `Text`) — the direct analogue of the glade demo's proven `NOTES`.
  Rationale: smallest convergent surface; a plain value (no record-map or log
  subtlety); requires no reshape of an existing per-tab plugin. *Zero-new-grip
  alternative:* bind the existing root `WORKSPACE_LIST`, accepting that a
  whole-array LWW value clobbers on concurrent edits to different records (single
  writer at a time is fine for Slice 0).
- **Scope:** single workspace `doc:W2`, single binder (no §4.4 yet). Effectively
  the standalone glade demo's wiring, hosted in gryth-ui.
- **Identity:** **STUB** via URL `?user=alice`/`?user=bob` → `stubGrant`.
- **Origin:** each tab needs a **distinct** origin. `demo/src/glade.ts`'s
  `stableOrigin()` persists `glade-origin` in `localStorage`, which is **shared
  across tabs of one browser profile** — so two same-profile tabs collapse to one
  participant chain. Setup precondition: either two browser profiles, or derive
  origin per tab (sessionStorage / the per-tab `tabContexts` id). Decide and
  document (open question 5).
- **Proves:** the gryth-ui shell hosts a glade-backed doc-scope grip; A's note
  appears on B's desktop; same code path as the glade demo.

### Slice 1 — private selection in the same workspace

Add `gryth:selection` (doc/private/value) as a doc-scope grip and bind it. Alice
and Bob each have their own selection in W2 (private zone) — proves **same share,
different key → isolated** inside one shared workspace. (Selection is per-user
here; a commons "shared focus" is a separate, optional surface — GDL-030, open
question 4.)

### Slice 2 — intersecting workspace sets (the headline)

Build workspace scope (§4.5) + M1 multi-binder (§4.4) + legibility (§4.6).
Grants: A `{W1, W2}`, B `{W2, W3}`. Then demonstrate **and assert**:

- A edits W2 notes → B sees it; B edits W2 → A sees it. **(commons converges)**
- A switches to W1, edits → B sees nothing; W1 absent from B's selector.
  **(A-only workspace isolated)**
- B switches to W3, edits → A sees nothing. **(symmetric)**

**New acceptance test (the de-risking this demo contributes):** one client
subscribes to `{doc:W1, doc:W2}`, the other to `{doc:W2, doc:W3}`, against the
real node; assert `doc:W1` ops are delivered to A and **never** to B (and
`doc:W3` never to A). This is the cross-share property that no current test
exercises (§2.1).

### Slice 3 — richer surfaces (optional, time-boxed)

- `gryth:wsmeta` / `WORKSPACE_LIST[W2]` repos/deps converge (workspace metadata).
- `TERMINAL_SESSIONS` for W2 converge as **records** (metadata only; live PTY
  output deferred, §7). Natural intersection: a W2 session shows on both desktops;
  a W1 session shows on A only.
- **If chat is wanted here:** it is *new work*, not a rebind. Chat is per-tab
  today (§2.6); to share it you must (i) introduce a root/per-workspace
  **log-shaped** transcript grip; (ii) define a `ChatLine`-shaped taut type +
  codec — gryth's `ChatMessage {id, role, text}` does **not** match the demo's
  `ChatLine {ts, user, text}` and carries no author, so `CODECS_BY_TYPE` cannot
  be reused verbatim and attribution needs a schema field; (iii) route the send
  handler through `binder.appendLog` (a `.update(prev => [...])` on a log atom
  produces no op, §2.3). Keep this fenced unless time allows.

### Demo-only shortcuts (fenced — NOT the real model)

| Shortcut | Real model | Where swapped |
|---|---|---|
| Identity via `?user=` URL param | Agent-issued, signed grant on auth | one line in `glade-sync` mints `grant` |
| `{self}` client-settable | `self` is an authenticated, non-forgeable claim | grant source only |
| Node serves any subscribe (M-LIMP) | Node would enforce `grant.allow` | **no enforcement work in scope** — node stays unenforced |
| In-memory client store | IndexedDB (same interface) | client-ts store |
| No encryption on private zones | Encryption for untrusted relays | additive; N/A on localhost |
| Workspace set hardcoded in URL/config | Workspace membership → grant `allow` | grant source only |
| Two browser tabs/profiles as "two desktops" | Two machines/users over network | transport URL only |

Every shortcut is isolated to **grant minting** or **transport config** — none
leak into surface/binder/component code.

---

## 6. Work items (ordered, each small)

Items 1–7 deliver Slice 0; 8–9 Slice 1; 10–13 Slice 2; 14 optional Slice 3.

1. **Resolve glade imports into the gryth-ui build (gates everything).** glade
   packages live outside gryth-ui's `packages/*` workspace and use *tree-relative*
   imports (`binder.ts` → `../../client-ts/src/session.ts`; `demo/src/glade.ts` →
   `../../../taut/corpus/glade.ir.json`). grip-share has **no `index.ts`/exports
   map**, and the glade IR schema (`../../glial-dev/taut/corpus/glade.ir.json`)
   plus the app taut IR (`../../glial-dev/glade/demo/ir/workspace.ir.json`) live
   outside `glade/`. **Decide the approach now:** vite aliases to the symlinked
   `src/` paths, or a thin re-export barrel inside `glade-sync`. Confirm vite
   resolves the symlinked TS (grip-share has no build step) and pin where the two
   `.ir.json` files get bundled.
2. **Create `@grythjs/glade-sync`** (peer to `plugin-api`). Exports
   `startGladeSync(grok, grant, opts)`; owns `Session`, `GripShareBinder`,
   `GladeClient`, callback wiring, subscribe loop, `dispose()`, and the
   `GLADE_STATUS` grip.
3. **Author `GRYTH_MANIFEST` + `CODECS_BY_TYPE`** in `glade-sync`, modeled on
   `demo/src/manifest.ts`. Slice 0 surface: `gryth:notes` (doc/commons/value/Text;
   `Text`/`Json` use the binder's default JSON codec — no taut codec needed yet).
4. **Add `stubGrant(user, docs[])`** in `glade-sync` — extend the demo's
   single-doc stub to mint one `allow` entry per workspace (`doc:{w}` keys
   `["", "self:{user}"]`) + `account:{user}`. Read `?user=` and the workspace set
   from URL/config.
5. **Add the `Doc.Notes` root doc-scope grip + tap** (`createAtomValueTap(DOC_NOTES,
   { initial:'', handleGrip:DOC_NOTES_TAP, share: surfaceDecl(GRYTH_MANIFEST,
   'gryth:notes') })`), registered at root in `src/taps.ts`. *(Or bind existing
   `WORKSPACE_LIST` per the Slice-0 alternative.)*
6. **Add a tiny notes view** (or reuse the workspace sidebar) reading
   `useGrip(DOC_NOTES)`, writing via the handle. No React state.
7. **Call `startGladeSync()` from `src/bootstrap.tsx`** after `registerAllTaps()`;
   `dispose()` on teardown; add a sidebar status dot bound to `GLADE_STATUS`.
   **Verify Slice 0:** two distinct-origin tabs (`?user=alice`, `?user=bob`), W2,
   notes converge. Must pass `scripts/no-react-state.test.mjs`.
8. **Add `gryth:selection`** (doc/private/value) to the manifest + a `Doc.Selection`
   root doc-scope grip with its share decl; wire a view (explorer or workspace
   viewer) to read/write it.
9. **Verify Slice 1:** alice's and bob's selections in W2 stay private; notes still
   converge.
10. **Add `WORKSPACE_ID` grip + per-workspace grok context** (`ws:{id}`, modeled
    on `tabContexts.ts`); register doc-scope taps within the active workspace
    context. *(Or the two-context fallback, §4.5.)*
11. **Add the writable sidebar workspace selector** (replaces the read-only
    `Desktop.tsx:703` label) + the identity badge + the shared-vs-private
    affordance (§4.6).
12. **Implement M1 multi-binder** in `glade-sync`: one `{Session, binder, client,
    scope}` per `doc:*` in `grant.allow`, each bound to that workspace's context.
13. **Verify Slice 2 + the new acceptance test (§5):** grants A`{W1,W2}` /
    B`{W2,W3}`; W2 converges; W1/W3 isolated and absent from the other user's
    selector; assert `doc:W1` ops never reach B.
14. *(optional)* Slice 3 surfaces (`gryth:wsmeta`, `TERMINAL_SESSIONS` records;
    shared chat only with the §5 caveats). Document PTY-output deferral.

---

## 7. Risks, open questions, REAL vs STUBBED

### REAL (proven; used as-is)

- Glade node multi-share routing by exact `(share, glade_id, key)` triple, minus
  origin; per-origin hash chains; deterministic `foldValue`/`foldLog`.
- One `Session`/one socket spans many shares; one node session subscribes to many
  triples.
- `GripShareBinder` generic bind/applyRemote/onLocalOps/appendLog/resync (grip-core-agnostic).
- `Manifest`/`Grant`/`manifestScope`/`surfaceDecl`/`manifestCodecs`.
- grip-core `ShareDecl` + `AtomValueTap` share hooks; `createAtomValueTap` accepts
  `share`; `useGrip` unchanged.
- **commons-converges / private-isolates within one share** — proven end-to-end in
  `glade/grip-share/test/node_integration.test.ts`.
- gryth-ui shell, plugin registry, per-tab contexts, no-React-state enforcement.

### NEW (must be built/tested by this demo)

- **Cross-share (disjoint workspace-set) isolation end-to-end** — guaranteed by
  the router but never exercised; Slice 2's acceptance test is its first proof.
- **Multi-workspace binding (M1)** — one stack per held workspace over
  per-workspace contexts; the binder's one-addr-per-tap limit is why M1, not M2.
- **Minimal per-workspace doc scope** (`WORKSPACE_ID` + `ws:{id}` context).
- **Legibility surfaces** (identity badge, writable selector, shared/private
  affordance) — none exist today.
- **`Doc.Notes`** root doc-scope grip for Slice 0 (chat is per-tab; not a rebind).

### STUBBED (demo-only, fenced in §5)

- Identity: URL → `stubGrant`; no signature/expiry; `self` client-settable.
- No client-side disk persistence (in-memory store).
- No encryption on private zones (localhost, trusted node).
- Node identity/capability fields stay **unenforced** (M-LIMP) — no enforcement
  hooks are built.
- Workspace membership hardcoded (URL/config), not a real ACL.

### Open questions (decide before/while executing)

1. **Multi-workspace binding (§4.4):** confirm M1 (binder-per-workspace, N sockets)
   for the demo; M2 deferred. — blocks Slice 2.
2. **Workspace scope (§4.5):** full `ws:{id}` context machinery vs the two-context
   fallback. Confirm the minimal version is acceptable as demo-only, not a
   `GrythVision` commitment.
3. **Slice-0 surface:** new `Doc.Notes` (recommended) vs rebinding `WORKSPACE_LIST`
   (zero new grips, LWW-clobber caveat).
4. **Selection semantics (GDL-030):** private-per-user (Slice 1 default) vs a
   commons shared-focus surface. *Recommendation: private; shared focus only if
   time allows.*
5. **Origin uniqueness:** two browser profiles vs per-tab origin override. Hard
   setup precondition for "two desktops" to be two participants in one machine.
6. **glade import portability (work item 1):** vite alias vs re-export barrel; and
   where the two `.ir.json` schemas bundle. Gates everything.
7. **Chat (Slice 3):** worth the new log grip + taut `ChatLine` codec + author
   field + `appendLog` wiring, or leave chat out of the demo?

### Known limitations to state, not fix

- Live PTY/terminal output (class-2 stream) is **not** modeled; the terminal demo
  shows session **records** converging only.
- Reconnect re-ships all ops (`resync()` via `session.dump()`); fine on localhost,
  not gap-filled.
- No equivocation/fork UI; `Session` throws are swallowed in `applyRemote`.

---

## 8. One-screen summary

- **Workspace = glade `share`.** Workspace set = the `doc:*` list in `grant.allow`.
  Intersection = set intersection of those strings — free, no ACL tree.
- **commons (`key=""`) converges; private (`key="self:<user>"`) isolates** — even
  inside one shared workspace.
- The **only new code** is a `@grythjs/glade-sync` package that attaches the proven
  `GripShareBinder` at bootstrap, a minimal per-workspace doc scope, a couple of
  doc-scope grips, and the legibility surfaces. Components are unchanged.
- The gotcha the prior analysis corrected: **chat is per-tab today**, so Slice 0
  spines on a new doc-scope `Doc.Notes` (or root `WORKSPACE_LIST`), not chat.
- Staging: Slice 0 notes converge (one workspace, two tabs) → Slice 1 private
  selection → Slice 2 the headline: A`{W1,W2}` ∩ B`{W2,W3}` = `{W2}` converges,
  `W1`/`W3` stay private — with the first end-to-end cross-share isolation test.
- Everything stubbed (identity, persistence, encryption, membership, node
  enforcement) is isolated to **grant minting** and **transport config** — none of
  it touches surfaces, the binder, or React components.
