# GripLab Grip & Tap Inventory

Status: reference snapshot + **proposed classification (draft)**.
Source: `grip-lab` @ `fb33d14` (branch `phase1-rust-iroh-spike`, 2026-06-04).

Purpose: complete inventory of the grips and taps in the grip-lab UI,
classified by tap class (per [GrythVision](GrythVision.md) persistence model:
**1** atom / **2** source value / **3** conversion) and by **proposed gryth
scope** (doc / environ / instance / *promotable* = environ input that may be
promoted to shared focus). Input to the gryth tap architecture.

## How the current wiring works

- One runtime singleton (`runtime.ts`); bootstrap picks
  `registerLabMockTaps()` vs `registerLabServiceTaps()` from `LAB_DATA_MODE`
  (`__GRIPLAB_RUNTIME__` global ?? `VITE_GL_DATA` ?? `'mock'`). The mock↔service
  seam is a **boot-time module branch**, not a grip-resolved binding (contrast
  grip-react-demo's `withOneOf(WEATHER_PROVIDER_NAME, …)` runtime bindings).
- `registerLabUiTaps()` registers **43 atom value taps**, one per class-1
  grip, each publishing a value grip + a `.Tap` handle grip.
- Mock mode adds: 3 workspace function taps (fakeData), GraphSim, mock file
  content, mock diff, mock session output.
- Service mode adds 12 service taps over the websocket `ServiceClient`,
  built on two grip-react primitives:
  `createAsyncMultiTap` (request/reply + TTL cache) and
  `createAsyncStreamMultiTap` (subscribe + retry/backoff + reset). An iroh
  variant of the terminal tap is selected by `VITE_GL_IROH_TERMINAL`.
- Per-view destinations: editor columns/diff panes run in child contexts and
  set destination params (`ACTIVE_FILE`, `FILE_REF`, `FILE_WINDOW`,
  `DIFF_WINDOW`); content taps key requests by `(destContext.id, params…)`.
- **Write path is imperative**: controls (`cmd.run`, `term.*`, chat post,
  `settings.update`, `admin.restart`) are `serviceClient/*` calls invoked
  directly from components. Reads are declarative; writes are not.

Grip-react primitives in use: `createAtomValueTap`, `createFunctionTap`,
`createAsyncMultiTap`, `createAsyncStreamMultiTap`, `BaseTap` (hand-rolled).

## Grip inventory

Every class-1 grip below also has a paired `*.Tap` handle grip (~45 of them):
runtime wiring, class 1 by shape, **never persisted** — omitted from the
tables.

### Navigation, appearance, personal prefs — class 1

| Grip | Value | Scope (proposed) | Notes |
| --- | --- | --- | --- |
| `Lab.CurrentView` | ViewId | environ, **promotable** | the stateUrl set |
| `Lab.Theme` | ThemeId | environ | |
| `Lab.UiScale` | UiScaleId | environ | |
| `Lab.WorkspaceLayout` | tiles\|graph | environ | |
| `Lab.PeerAvatars` | overrides map | environ | personal display prefs, not doc identity |

### Panel geometry & transient UI — class 1

| Grip | Scope (proposed) | Notes |
| --- | --- | --- |
| `Lab.ChatPanelOpen`, `Lab.ChatPanelWidth`, `Lab.ChatComposerH` | environ | desktop-document geometry |
| `Lab.ExplorerOpen`, `Lab.ExplorerWidth`, `Lab.ExplorerCollapsed` | environ | |
| `Lab.DiffFilePickerCollapsed` | environ | |
| `Lab.ChatPanelDragging`, `Lab.ChatComposerDrag`, `Lab.ExplorerDrag` | **instance** | drag-in-progress, never replicate |
| `Lab.WorkspaceMenu`, `Lab.RunReposOpen`, `Lab.RunDialogOpen`, `Lab.DiffFilePickerOpen` | **instance** | open menu/dialog state |

### Selections, focus, drafts — class 1, the promotable middle

| Grip | Scope (proposed) | Notes |
| --- | --- | --- |
| `Lab.SelectedPeerId` | environ, **promotable** | routes most source taps (home param) |
| `Lab.SelectedFile` | environ, **promotable** | stateUrl set |
| `Lab.FocusLine` | environ, **promotable** | the google-docs cursor analog |
| `Lab.SelectedSession`, `Lab.SelectedTarget` | environ, **promotable** | |
| `Lab.DiffLeft`, `Lab.DiffRight` | environ, **promotable** | stateUrl set |
| `Lab.EditorGroups`, `Lab.ActiveGroup` | environ | seed of the desktop window/split model |
| `Lab.SessionSearch`, `Lab.SessionFilters` | environ | |
| `Lab.ChatDraft`, `Lab.SessionDraft`, `Lab.ChatPending` | environ | drafts roam |
| `Lab.RunRepos`, `Lab.PurgeDays` | environ | |
| `Lab.OnboardingForm`, `Lab.CollabEdit`, `Lab.AvatarEdit`, `Lab.PeerHealthDialog` | instance (dialog-ish) | judgment call; forms could be environ |

### Per-destination view params — class 1, structural

| Grip | Scope (proposed) | Notes |
| --- | --- | --- |
| `Lab.View.ActiveFile`, `Lab.FileRef`, `Lab.View.FileWindow` | environ (per window) | per-editor-column dest params → gryth per-window facet bindings |
| `Lab.DiffWindow` | environ (per window) | |

### Workspace ignore filters — class 1 + class 2 (dual-provided)

| Grip | Notes |
| --- | --- |
| `Lab.FileIgnore.Enabled`, `.Patterns`, `.Text` | atoms registered **and** service `settings.get` tap provides the same grips in service mode; `.Text` is the editable draft of `.Patterns` |

### Doc-scope shared state — class 2 (atom-provided in mock)

| Grip | Upstream (service mode) | Notes |
| --- | --- | --- |
| `Lab.Peers` | `peer.presence.subscribe` | atom suppressed in service mode (only grip with explicit suppression) |
| `Lab.ChatMessages` | `chat.subscribe` | full-list snapshots; **atom still registered too** |
| `Lab.Sessions` | `sessions.subscribe` (peer-routed) | **atom still registered too** |
| `Lab.WorkspaceRepos` | `workspace.status.subscribe` (peer-routed) | mock: function tap over fakeData |
| `Lab.WorkspaceDependencyEdges` | `deps.get` (TTL 30 s) | mock: derived from repos (class 3) |
| `Lab.WorkspaceTree`, `…TreeVersion`, `…TreeStatus` | `tree.subscribe` (peer-routed) | status grip is the ad-hoc envelope |

### Class-2 outputs (cache of upstream) and class-3 derivations

| Grip | Class | Producer | Notes |
| --- | --- | --- | --- |
| `Lab.View.FileContent`, `…FileGitStatus`, `…FileLineIndex` | 2 | file tap (windowed delta protocol, `TextWindowReassembler`) | mock: class 3 over fakeData |
| `Lab.View.FileStreamStatus` | 2-status | file tap | ad-hoc status envelope |
| `Lab.Service.DiffHunks`, `…DiffDiagnostics`, `…DiffVersion`, `…DiffStreamStatus` | 2 | `diff.subscribe` (server-computed) | mock: genuine class-3 local `lineDiff` |
| `Lab.SessionOutput`, `Lab.SessionOutputSource` | 2 | ws snapshot stream **or** iroh byte stream | single accumulated string — no log/cursor semantics |
| `Lab.SessionDiagnostics` | 3 | `parseDiagnostics()` inline in each output tap | conversion baked into source taps (duplicated) |
| `Lab.GraphNodes` | 3 | GraphSimTap (rAF physics) | presentation-only; instance scope |
| `Lab.Service.Connection` | 2-status | `watchStatus` | **instance** scope — per-client connection |

Counts: 60 grips in `grips.ts` + 6 in `grips.service.ts` (+ ~45 `.Tap`
handle grips).

## Tap inventory

### Always registered (both modes)

| Tap | Primitive | Provides | Params | Class |
| --- | --- | --- | --- | --- |
| 43 × UI atoms (`registerLabUiTaps`) | `createAtomValueTap` | each class-1 grip + handle | — | 1 |
| GraphSimTap | `BaseTap` (custom) | `GRAPH_NODES` | engine input via `graphEngine.setInput()` from components; hover/pin/drag held in module vars | 3 |

### Mock mode (`registerLabMockTaps`)

| Tap | Primitive | Provides | Params | Notes |
| --- | --- | --- | --- | --- |
| MockWorkspaceRepos | function | `WORKSPACE_REPOS` | home: `SELECTED_PEER_ID` | fakeData by peer |
| MockDepEdges | function | `WORKSPACE_DEP_EDGES` | home: `WORKSPACE_REPOS` | derived |
| MockTree | function | `WORKSPACE_TREE`, `…VERSION` | — | static |
| MockFileContent | function | `FILE_CONTENT`, `FILE_GIT_STATUS` | dest: `ACTIVE_FILE`, `FILE_REF`; home: `SELECTED_PEER_ID`, `PEERS` | per-column dest contexts; "the seam where the real source plugs in" |
| MockDiffContent | function | 4 diff grips | dest: `ACTIVE_FILE`, `DIFF_LEFT/RIGHT`, `DIFF_WINDOW` | real local `lineDiff` — genuine class 3 |
| MockSessionOutput | function | `SESSION_OUTPUT`, `SESSION_DIAGNOSTICS` | home: `SESSIONS`, `SELECTED_SESSION`, `SELECTED_TARGET` | reads output out of mock `SESSIONS` rows |

### Service mode (`registerLabServiceTaps`)

| Tap | Primitive | Provides | Key params | Upstream |
| --- | --- | --- | --- | --- |
| ServiceState | stream | `SERVICE_CONNECTION` | — | `client.watchStatus` |
| ServiceSettings | req/reply (TTL 10 s) | `FILE_IGNORE_*` | — | `settings.get` |
| ServiceWorkspaceStatus | stream | `WORKSPACE_REPOS` | home: `SELECTED_PEER_ID` | `workspace.status.subscribe` (hub-routable) |
| ServiceDepsGraph | req/reply (TTL 30 s) | `WORKSPACE_DEP_EDGES` | home: `SELECTED_PEER_ID` | `deps.get` |
| ServiceTree | stream | tree + version + status | home: `SELECTED_PEER_ID` | `tree.subscribe`; reset→loading |
| ServiceFileContent | stream | 4 file grips | dest: `ACTIVE_FILE`, `FILE_REF`, `FILE_WINDOW`; home: `SELECTED_PEER_ID` | `file.subscribe` windowed snapshot/delta/reset; module-level reassembler map |
| ServiceDiffContent | stream | 4 diff grips | home: `SELECTED_FILE`, `DIFF_LEFT/RIGHT`; dest: `ACTIVE_FILE`, `DIFF_WINDOW` | `diff.subscribe` |
| ServiceSessions | stream | `SESSIONS` | home: `SELECTED_PEER_ID` | `sessions.subscribe` (routed) |
| ServiceSessionOutput | stream | output + source + diagnostics | home: `SESSIONS`, `SELECTED_SESSION`, `SELECTED_TARGET` | `session.output.subscribe` full snapshots; peer inferred from `SESSIONS` list |
| IrohSessionOutput | stream | same 3 | same | `term.ticket` → wasm `subscribe_terminal`; accumulates bytes → snapshot; no cancel, unbounded buffer (flagged in-source) |
| ServiceChatMessages | stream | `CHAT_MESSAGES` | — | `chat.subscribe` full lists |
| ServicePeers | stream | `PEERS` | — | `peer.presence.subscribe` |

Retry/backoff: shared `SERVICE_STREAM_RETRY` const (500 ms → 5 s, ×2,
jitter 0.4). Peer routing: a private `routePeer()` helper duplicated in 4
taps.

### Outside the tap system

- **Controls**: `serviceClient/{commands,chat,settings,admin}.ts` — invoked
  imperatively from components (run command, terminal input/resize/close,
  post chat, restart hub/client).
- **terminalController.ts** — imperative xterm wrapper (ref-callback mount,
  prefix-append write optimization, mock echo mode).
- **Hidden state**: graphEngine module vars (positions, hover, pin, drag),
  `TextWindowReassembler` map (cleared only on error), iroh debug probe
  global, xterm buffers.

## Findings for the gryth tap architecture

1. **Atom boilerplate.** Every class-1 grip costs 4 hand-written artifacts
   (grip, `.Tap` grip, default, registration line) × 43. Declare atom groups
   once (schema → generated grips + registration + environ persistence tag).
   This is the denormalized-DB-of-code smell, and it is also exactly the
   environ-document declaration: scope tags belong in the same schema.
2. **Mode seam is boot-time, not resolved.** `dataMode` branches module
   registration; the demo's `withOneOf` binding pattern is strictly better:
   per-surface provider selection in graph resolution, runtime-swappable,
   mixable (mock chat + real terminal in one session).
3. **Implicit ownership / double provision.** Service mode registers UI
   atoms *and* service taps for `SESSIONS`, `CHAT_MESSAGES`, `FILE_IGNORE_*`
   (only `PEERS` is explicitly suppressed); the winner falls out of resolver
   scoring. In gryth, scope + authority (who owns this grip) must be
   declared, not emergent.
4. **Reads declarative, writes imperative.** All controls bypass the graph
   as ad-hoc client calls from components. Gryth needs declared
   exchanges/controls (vision C1) carrying capability + `(principal,
   on_behalf_of)` — this is where attribution physically lands.
5. **Terminal output has no flow type.** Both ws and iroh paths accumulate
   one unbounded string and re-snapshot it. The glade terminal slice shape
   (live stream + replay log + cursor + materialized screen, keyed by one
   source handle) is the fix; `SESSION_OUTPUT_SOURCE` is a proto-handle.
6. **Status envelopes are ad hoc.** `FILE_STREAM_STATUS`,
   `WORKSPACE_TREE_STATUS`, `DIFF_STREAM_STATUS`, `SERVICE_CONNECTION` are
   four hand-rolled shapes for the same concept. Standardize one source
   status envelope (idle/loading/ready/error/stale + provenance) — GDL-003
   client-side.
7. **Routing logic is copy-pasted.** `routePeer()` ×4, string-concat request
   keys per tap, peer inferred by scanning `SESSIONS`. A declared source
   handle should carry the route key (GDL-026; GripShareAdvertisement
   already flags the inference as unreliable).
8. **Source-tap shape is generative.** The 12 service taps are one pattern
   instantiated by hand: (provides, params→request key, subscribe|fetch,
   mapEvent, initial, retry). That tuple is data — a declared surface
   binding should generate the tap (config-as-data), which is also exactly
   what a `.glade`-described surface needs at the client end.
9. **Derivations buried in source taps.** `parseDiagnostics` runs inside
   both output taps; mock diff computes locally while service diff is
   server-computed. Class-3 conversions should be standalone function taps
   over class-2 grips so the substrate choice (local vs server compute)
   stays behind the seam.
10. **Hidden state should be named.** Graph pin/hover, reassembler caches,
    xterm buffers are legitimate instance-scope presentation state — but
    they live in module vars. Gryth should at least classify them
    (instance scope, class 2/3 cache) so authoritative state never hides
    there.
11. **Mock data is baked into declarations.** `PEERS`/`SESSIONS`/`CHAT`
    defaults come from `fakeData` in `grips.ts`. Grip declarations should
    carry neutral defaults; mocks provide values through taps like any other
    provider.
12. **Per-destination contexts work.** The editor-column pattern (child
    context + dest params, request keyed by `destContext.id`) is the proven
    seed of the desktop model's per-window facet bindings — keep it, and
    formalize it in the window/tile schema.
