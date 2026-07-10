# Gryth Vision

Status: **draft** — to be fleshed out, then prioritized/pruned for development.

Purpose: the unified vision for gryth — the productized GripLab successor
(new UI + new protocol) on the grip/glade/glial stack. This document states
what gryth is and the commitments that unify it; it deliberately does not yet
cut scope.

## One sentence

Gryth is a shared dev workspace where every surface is declared, flow-typed
data; each participant's UI is a durable **desktop** — windows over those
surfaces — that persists and roams across their instances; and humans and AI
agents are peers distinguished only by capability and attribution.

## The google-docs model (three tiers)

- **doc scope** — the shared artifact: workspace state, terminals, chat,
  VM inventory, the activity graph. Durable, multi-writer, attributed.
- **environ scope** — the user's desktop: window layout, focus, selections,
  filters. Durable, owned by one user, replicated across all instances of
  that user's environ (laptop, desktop, second browser), and delegable —
  this is the scope an agent writes to drive the UI.
- **instance scope** — one running client: viewport, drag-in-progress,
  transient hover/menu state. Ephemeral, never replicated.

What earlier drafts called "session scope" was conflating the last two; the
desktop model splits them. The grip-lab inventory confirms the split is real:
`Lab.CurrentView` is environ, `Lab.ChatComposerDrag` is instance.

North star: an agent holding an environ capability can drive a user's UI —
"open the debugger for me" is a capability grant plus a write to a declared
environ surface, not a feature.

## Three commitments

### C1 — Everything on screen is a declared surface (doc scope)

Terminals (source → live stream + replay log), diffs (derived view over two
declared endpoints), the commit/activity graph (typed nodes and edges), the
VM inventory (machine_config / base / instance / volume), chat (append log).
Consumers never know the producer; mock → real is a provider swap.

Consequence: a button in the UI and an MCP agent invoke the **same declared
exchange** with the same capability check. "1-click VM" and "an agent can do
it" are one feature, not two.

### C2 — The UI is a desktop, not tabs (environ scope)

The shell is a window manager in the macOS/Windows/GNOME/KDE sense. Each
window is an individual view instance — a chat, a vm-manager, a workspace
view, a terminal — and the same kind can be open more than once. Windows
float, stack, tile side-by-side, minimize, and carry z-order; tiling/splits
can live inside a window where a view wants panes.

The desktop itself is data: an environ-scoped document (windows, geometry,
z-order, focus, per-window facet binding to a source handle) that persists
and replicates across all instances of the user's environ. Stacking
terminals, side-by-side diffs, and mix-and-match views are properties of
this, not features.

Because the desktop is data it is: shareable (the `stateUrl.ts`
generalization), followable (presenter mode), and AI-drivable (a delegated
principal edits my desktop document).

Two seams the WM metaphor makes explicit:

- **window lifecycle ≠ source lifecycle.** Closing a terminal window detaches
  a view; the PTY source and its log live in doc scope. Re-opening binds a
  new window to the same handle.
- **launcher = advertised facets.** "What can I open?" is the palette of
  declared surfaces the user holds capabilities for.

Position (tentative): buy what's buyable for window/dock mechanics, own the
desktop data model. The differentiator is the desktop being grip/glade data,
not the drag mechanics.

### C3 — Humans and agents are the same kind of participant (identity)

One protocol door for all principals. Agents enter via the MCP endpoint under
an explicit delegation; every write carries **(principal, on_behalf_of)** in
the record envelope — attribution is protocol, not UI convention. Chat is the
principal-to-principal spine and, with surface references linkable into it,
the team's intent trail. AI-produced findings carry provenance and staleness
(per GLConceptVisualizer) — the same discipline as write attribution.

## Persistence model (grip tap classes)

Grip has three tap classes, and they decide what persistence exists at all:

1. **Atom / multi-atom** — application-context state: a path and a value
   ("current tab", "weather location"). Authoritative where it lives.
   In theory this is all you need — everything else takes these as
   parameters and fetches, converts, or displays.
2. **Source value** — the cached latest value of a query (file content,
   terminal output). The local copy is cache, not truth; it can always be
   refetched from the source.
3. **Conversion (function)** — pure data-in/data-out. Nothing stored.

Consequences:

- **Only class 1 needs persisting on the client.** The environ document is
  precisely the *serializable* class-1 atom map — `stateUrl.ts` made
  general. The non-serializable class-1 grips (tap handles, function grips)
  are runtime wiring: rebuilt on boot, never persisted. Sharing a session
  across browsers or headless clients = replicating this map, nothing more.
- **Persistence is tap class × scope.** Instance-scope atoms (drag state)
  are class 1 by shape but never replicated. Promoted atoms (shared focus)
  are class 1 in doc scope, persisted by the share itself. Doc-scope truth
  behind class 2 (terminal logs, chat) is the provider's durability problem,
  not the client's.
- **Class 2 is cached locally for snappiness**, with status/refetch
  semantics (cf. `Lab.View.FileStreamStatus`, the retry taps). A replay
  cursor into a class-2 log is itself class-1 session state (the GDL-028
  cursor question, client-side).
- Sharing class-2 caches *between* clients is real but deferred — that is
  the p2p story. Don't rule it out; don't build it now. The seam already
  permits it (a peer cache is just another provider behind the tap).

Convergence check: class 1/2/3 ≈ the Grip Share advertisement roles
`input` / `output` / derived — two documents reached the same split
independently.

## The vision points, mapped

| # | Point | Commitment | Notes |
| --- | --- | --- | --- |
| 1 | Chat — work with the team, AI and human | C3 | spine + intent trail; surface links (ChatLink generalized) |
| 2 | Workspace view — commit activity at a glance, dig quickly | C1 | timeline/hotspot lens first; full concept graph later behind the same seam |
| 3 | VM manager — new VM workspace in 1 click, or an agent can | C1 + C3 | VmMgrDataModel entities as surfaces; rebase/create as exchanges |
| 4 | Agents identified "on behalf of" via MCP endpoints | C3 | delegation + attribution in the envelope (GDL-004) |
| 5 | Terminals stack and drag around | C2 | tiles in stack containers |
| 6 | Desktop / window manager — windows as view instances; environ persists and roams across the user's instances | C2 | environ scope; window lifecycle ≠ source lifecycle |
| 7 | Diffs pulled side by side | C2 | two diff windows/tiles in a row; diff endpoints are declared inputs |
| 8 | Tabs too rigid — mix and match | C2 | desktop-as-data kills fixed views at the model level |

## What existing artifacts contribute

- `grip-lab/src/lab/grips.ts` — proof the UI state can be 100% data
  (enforced by the no-react-state test). The inventory splits cleanly into
  doc / environ / instance / promotable-middle.
- `grip-lab/src/lab/stateUrl.ts` — a view as a grip→value map; the embryo of
  layout-as-data and shared focus. Its six keys are the empirically
  discovered promotable set.
- `glial-dev/dev-docs/examples/griplab/*.glade` — the facet list is the tile
  palette; the terminal slice (source → live_channel / stream / log /
  materialized) is the C1 reference shape.
- `grip-lab/dev-docs/GLConceptVisualizer.md` — data model for point 2 (typed
  nodes/edges, deterministic-first, AI annotations with provenance and
  staleness). Its adversarial section governs scope.
- `vm_manager/dev-docs/VmMgrDataModel.md` — data model for point 3; already
  declarative-disciplined (immutable config snapshots, repointable base,
  durable volumes; "nothing precious lives in the overlay").
- `glial-dev/dev-docs/grip-share/GripShareAdvertisement.md` — surface roles
  (input/output/control/…) and visibility (private/session/shared/delegated).
- Open decisions this vision lands in: `GDL-030` / `GSA-OQ-003`
  (session-local vs shared inputs, promotion), `GDL-004` (delegated
  references), `GDL-020` (canonical schema across Py/TS — forced before DSL
  aesthetics once the protocol is a real boundary).

## Design positions (tentative — revisit at prune time)

1. UI first against mock taps (`gryth-ui` scaffold exists); the protocol
   surface inventory becomes the contract the mocks implement.
2. Substrate v0 behind the seam can be a conventional service; p2p
   (iroh single-exe, razel, file monitor) arrives later behind the same seam.
3. Timeline/commit-activity lens before the full concept graph.
4. Buy window/dock mechanics where buyable, own the desktop data model.
5. Attribution shape `(principal, on_behalf_of)` is a protocol-envelope
   requirement from day one, even while auth itself is stubbed.

## Open questions (flesh-out list)

- Desktop document schema: windows, geometry, z-order, stacks/splits, facet
  bindings, focus — needs the same declaration rigor as protocol surfaces.
  Per-window context params (cf. `Lab.View.*` destination params) are the
  Grip-model refinement candidate.
- Mirrored vs independent: do two live instances of one environ render the
  same desktop (move a window here, it moves there), or separate desktops
  over shared content (virtual desktops / KDE activities per environ)?
  Likely both, via a per-instance "current desktop" pointer.
- Concurrent instances: conflict semantics when two clients edit the desktop
  at once — per-window geometry as LWW atoms is probably enough; focus may
  need to be per-instance.
- Promotion semantics: my selection → shared focus; follow/presenter mode;
  which environ surfaces are promotable by default.
- Delegation UX: how a human grants/revokes an agent's environ capability;
  how "on behalf of" renders in chat, history, and the activity graph.
- Chat scope: message log vs intent trail — how far do surface links,
  AI findings, and exchange receipts go into chat?
- VM workspace join: what exactly happens between "instance running" and
  "workspace participating" (mount, identity, advertisement)?
- What of the grip-lab UI survives: which views port as facets vs get
  redesigned vs get dropped?

## First slice (sketch only — prune before building)

Minimal desktop shell (window mechanics + environ-scope desktop grips,
mock-persisted) hosting three windows over mock taps: chat, one terminal,
one diff. Then one real surface (terminal) behind the seam, and desktop
persistence/roaming behind the same seam. Proves C2 end-to-end, C1 on one
surface, and the environ-scope delegation path an agent would use.
