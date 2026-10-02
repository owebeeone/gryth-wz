# Settings and session desks

Status: implemented; verification recorded below, 2026-10-02. Work belongs in the `appearance`
lane. Owner rulings: `GrythGripScopes.md` §3.4, all eight recommendations
accepted on 2026-10-02. The existing Glial appearance plan remains authoritative
for appearance; the cross-node writes plan remains authoritative for transport.

## Contract and boundaries

- SS-01: appearance MUST follow the principal across sessions on the Gyld desk.
- SS-02: a named principal and `?session=` MUST select one desk across pages.
  An omitted session MUST use a browser-persisted name per entry and principal.
  An unnamed principal MUST retain the browser-only desk.
- SS-03: one `value`, `<entry>.desk`, on `ws-razel`, private key
  `self:<principal>/<session>`, MUST hold the six layout values (12 grips).
  Writes MUST debounce for 300 ms. This is whole-document last-writer-wins;
  concurrent unrelated edits MAY overwrite one another.
- SS-04: first paint MUST use stored session state when available. Replay MUST
  precede migration writes; an existing remote desk MUST win over the legacy
  blob. A legacy desk MUST seed an empty session only once. A refused replay
  MUST NOT trigger migration. Local ops MUST survive reload through IndexedDB; explicit edits made before
  successful replay MUST also remain in the browser session cache for retry.
- SS-05: current desktop MUST be shared; keyboard focus, reveal cues, cameras,
  drafts, hover and gestures MUST stay per page. Reveal expiry MUST NOT write
  the shared desk.
- SS-06: changed tab links MUST update live destination grips; local picks MUST
  update the tab record. Reapplication MUST preserve contexts and instance state,
  and wired sinks MUST continue inheriting their source.
- SS-07: Gyld set roots and semantic focus MUST live in the session document,
  retaining their future doc-promotion classification. Directory handles MUST
  remain runtime-only. This work does not build promotion to a team document.
- SS-08: settings MUST name the session and provide a link to open it elsewhere.
  Reset layout MUST reset that session while preserving appearance.

The desktop document codec and synchronization state machine are pure modules
with injected storage, clock, desk ports and zone controller. Their contract
states read/write/watch, replay outcome, reset and disposal behavior. The
settings live adapter is integration: it owns the existing Glial binder,
IndexedDB engine and Glade destination. No new dependency is required. The
plugin API adds a small tab-link field contract so the desktop does not import
concrete plugins. Existing packages retain their roles; no allowlist changes.

## Phase 1 — scope and page-local cues

1. Update GrythVision and coding scope comments against GDL-030.
2. Write regression tests proving reveals never mark the shared record and
   expiry changes only a page-local grip. Move the cue to that grip; preserve
   raise, active-tab selection and focus behavior.

## Phase 2 — naming and persistent session document

1. Test URL naming, generated browser defaults, principal/entry isolation,
   blocked storage and encoded links; implement session resolution.
2. Test an injected session-layout controller: replay before seed, remote
   changes without echoes, debounced local edits, stale timer cancellation,
   corrupt input, one-time migration, denied replay, reset and disposal.
3. Wire `<entry>.desk` into the Gyld composition with a distinct binder instance in the existing IndexedDB engine;
   provide a persistence factory through DesktopSetup. Keep the existing local
   persistence path for the full desktop and unnamed users.
4. Add the settings session label/link. Declare `gyld.desk` in the Gyld app;
   update the binding census test before the declaration change.

## Phase 3 — live tab records and Gyld session state

1. Test the tab-link field contract with a real grip context: both directions,
   stable context, preserved instance atoms, removed fields, wired inheritance
   and cleanup. Implement it; cover affected plugin consumers.
2. Bind the 24 destination/legend/conversation grips from the scope inventory,
   plus chat group, workspace selection and explorer WTA.
3. Persist serializable Gyld set roots and semantic focus through injected
   session fields. Directory handles remain local and need reacquisition.

## Phase 4 — verification and delivery

Focused Vitest suites are the minor-edit loop. At the settled tree run `pnpm
lint`, `pnpm test`, `pnpm build`, and `pnpm build:gyld` in the lane. Run the
syntax-aware brace scan over changed TypeScript, including tests. Declaration
changes run the node binding census and grazel's affected consumer checks.

Tests MUST trace SS-01–08, including two independent page runtimes connected
through real client-ts sessions and Glial destinations. The final integration
check SHOULD use two isolated nodes on this Mac, separate scratch data and
ports outside 5173/8080/9099. It MUST NOT restart or write the owner's desk.
The two-machine run remains deferred. Report any unrun browser/node evidence
explicitly. Commit through gwz when a verified step is complete; do not push,
publish, merge to the live desk, or restart it as part of this request.

## Decision record

GDL-030 scope refinement (owner, 2026-10-02): environ is appearance; session is
one mirrored desk; instance is one page. Current desktop is session, keyboard
focus is instance. The whole document remains LWW until a measured lost move
justifies per-window values. Promotion UX and private-key authentication stay
with their existing plans, not this implementation.

## Implementation status

Phases 1–3 are implemented in the appearance lane. The persistence adapter
reuses the existing `gryth.appearance` IndexedDB engine with a separate desk
binder instance; appearance and desk remain separate Glade surfaces and keys.
The full desktop and unnamed users retain browser-local layout persistence.
No runtime dependency or library classification changed. New and modified
control-flow bodies are checked with the TypeScript parser; broader pre-existing
brace debt is not claimed as migrated.

### Requirement evidence

| Requirement | Tests |
| --- | --- |
| SS-01, principal/session isolation | settings `sessionDesk.test.ts`, real-node check |
| SS-02, names/blocked storage/encoded links | settings `session.test.ts` |
| SS-03, debounce/remote echo/stale timer | desktop `sessionLayout.test.ts`, node `binding_census.rs` |
| SS-04, replay/migration/refusal/offline reload | desktop `sessionLayout.test.ts`, settings `sessionDesk.test.ts` |
| SS-05, local cues/focus | desktop `attention.test.ts`, `reveal.test.ts`, real-node check |
| SS-06, both directions/lifetime/wires/codecs | desktop `tabContexts.test.ts`, Gyld `sessionLinks.test.ts`, `browser/destination.test.ts`, real-node check |
| SS-07, roots/focus/handle exclusion | Gyld `sessionState.test.ts`, real-node check |
| SS-08, session link/reset | settings `session.test.ts`, `sessionDesk.test.ts`, real-node check |

Red-first checks found and pinned the missing private desk declaration, live
record updates, page-local reveal ownership, a delayed own-value notice race,
loss of semantic ref on legend/perspective picks, and two offline recovery
races. Pending explicit edits now take precedence over stale local engine
state and inbound replay until successful replay permits their write.

### Settled-tree verification

- `pnpm lint`: passed.
- `pnpm test`: 76 files, 1,065 tests passed; the no-React-state scan passed.
- `pnpm build` and `pnpm build:gyld`: both passed. Existing
  Vite chunk-size notices remain; no thresholds were relaxed.
- TypeScript parser scan: 43 changed/new source and test files, zero unbraced
  modified bodies, including the dedicated integration test and config.
- `glade/node/check.sh`: all 9 components passed, 492 node-workspace tests
  for each composition root. Existing fmt/clippy debt stayed at its baselines:
  node 252 fmt hunks / 9 warnings; wire 1 hunk / 7 warnings. No allowlist changes.
- Binding census: 6 passed after the declaration change (6 failed before it).
- Grazel affected consumers: 30 + 3 + 5 + 1 passed, zero ignored tests and no
  test skipped. Its first boot reported the optional gwz supplier absent;
  the later composition tests built and exercised both suppliers. A deliberately
  empty Gyld checkout produced the expected refused-build diagnostic.
- Dedicated real-node check: passed on two isolated assembled nodes, each
  declaring `ws-razel`, with writes in both directions, session isolation,
  appearance roaming, page-local state, reload and late join. It uses independent
  module/page runtimes and actual WebSocket clients, without mocking transport.

The slow check is deliberately outside the normal Vitest include set. Run it
against two isolated nodes that both load the updated Gyld declaration, trust
one another's endpoints and grant the peer node `read.*,write.*` on `ws-razel`:

```sh
SESSION_NODE_A=ws://127.0.0.1:<scratch-port-a> \
SESSION_NODE_B=ws://127.0.0.1:<scratch-port-b> \
pnpm exec vitest run --config vite.session-check.config.ts
```

Both endpoints MUST be scratch instances, outside the desk's reserved ports.
This verifies headless UI runtimes and real node routing; it is not a rendered
browser interaction check. The two-machine run remains deferred. Team-document
promotion, auto reconnect after startup failure, and per-window merge policies
remain outside this implementation.

### Delivery commits

- `gryth-ui` `caf4965`: session desks, live tab links, local attention, tests and
  consumer documentation.
- `grazel` `a9f8231`: the private `gyld.desk` app binding.
- `glade` `90e852a`: binding census/migration regression tests.

Workspace lock updates are committed separately. Nothing was pushed or merged
into the owner's live UI checkout. The owner must integrate the appearance lane
and rebuild/restart the desk; its current binary predates cross-node writes.
Both two-node runs stopped only their own PIDs and deleted their homes/targets.
