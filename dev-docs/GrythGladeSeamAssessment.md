# Gryth ↔ glade/glial seam assessment

Date: 2026-07-11
Status: **assessment** — input for the gryth↔glade coordination effort.
Method: two parallel code/doc surveys run 2026-07-11 — one over `gryth-wz`
(assumption inventory), one over `glade-wz` (implementation reality). All
claims below cite the surveyed tree; dates are doc/commit dates.

## TL;DR

- The attachment mechanism gryth planned around — `GripShareBinder` +
  `@grythjs/glade-sync` (`GrythDemoProposal.md` §4) — **no longer exists**.
  The binder was deleted on the glade side 2026-07-10 (GC-3 cutover). The
  canonical seam is now GDL-035's compile wall:
  `grip-core ──▶ glade-decl ◀── glial`, with consumers attaching through
  **glial mounts** (`GlialTap` + declarative `FillSpec`,
  `glade-wz/glial/src/grip/index.ts`).
- glade/glial is **further along than gryth assumed on the spine** — a real
  Rust glade node (WS + iroh p2p, store/router/registry, 45 tests + E2E),
  a glial client kernel with IndexedDB persistence and reload-resume (44
  tests, GAP-9 closed 07-11), and a working multi-tab converging demo that
  is literally titled "Gryth Workspace Demo" — and **behind on everything
  gryth-specific**: no workspace-directory build (ratified GDL-032,
  unbuilt), no pty/file/chat/VM providers (the only "pty" in the tree is an
  echo-stub string).
- gryth's apparent doc contradiction (matcher flips vs binder swap — two
  competing "real provider" stories) **resolves itself**: the binder story
  is dead; the matcher flip is the gryth-side *selector* and a
  GlialTap-backed tap is the real provider *implementation* behind it. The
  two mechanisms compose; they were never rivals.
- The stable thing to build against today is **`glade-decl`** (contract
  `ccdae14`, byte-parity corpus across ts/py/rs, no churn since 07-07).
  `gryth-wz` already vendors `glade-decl-ts` as a workspace member at the
  current head (`4467e31`, post-bigint regen), and grip-core already rides
  it (`share_decl.ts`, types-only import).

## What broke (assumption → reality)

| # | Gryth assumed (source) | Reality (glade-wz, 2026-07-11) |
|---|---|---|
| 1 | Integration lands as `GripShareBinder` + `Manifest`/`Grant` + `GladeClient` WS, wrapped in `@grythjs/glade-sync` (`GrythDemoProposal.md` §4.1–4.2) | Binder **deleted** 07-10 (GC-3). grip-share shrank to declaration plumbing (`decl.ts`, `manifest.ts`). Consumers go through glial mounts (`GlialTap`/`FillSpec`). The demo-proposal integration plan (§4) is void. |
| 2 | glial = thin client runtime under grip (`StackMap.md`) | glial is now a **kernel** in its own right (GDL-035, ratified-on-their-log 07-07): local-persistence-first, refcounted binding instances, oracle-gated folds, IndexedDB engine, reload-resume. It is the mandatory middle layer, not a shim. |
| 3 | Live glade/glial docs are readable via `dev-docs/{glade,grip-share}`, `DecisionLog.md`, `StackMap.md` | Those are **symlinks into `~/limbo/glial-dev`**, which froze 2026-06-17 and was superseded by `glade-wz` on 07-07. Gryth has been reading a dead workspace; the live docs/decision log are `glade-wz/dev-docs/GladeProgramStatus.md` + `glade-wz/dev-docs/DecisionLog.md` (GDL-031…039) + `glade-wz/glade/dev-docs/`. |
| 4 | glade-decl-ts type surface as of ~06 | i64 fields (`ttl_ms/seq/base_seq`) became **`bigint`** with typed Decode/EncodeError (regen 07-08). Wire bytes unchanged. `gryth-wz`'s vendored copy is already at this head. |
| 5 | Sync model as sketched in `GrythDemoProposal.md` §2 (June) | s-sync reframed to per-`(origin, zone)` chains (D8/AZ-16/17, 07-10); zones carried on heads/ranges/tamper slots. The §2 triple `(share, glade_id, key)` survives, but June-era sync details are stale. |
| 6 | Base glade knows about the workspace app | GDL-037/038 (07-07): glade is now **app-agnostic substrate**; app endpoints are declared in `<app>.glade` data files. gryth's app is named **grazel** — and `grazel-app.glade` does not exist yet. |

## What held

- **The compile wall (GDL-035)** — grip-core's `ShareDecl` rides
  `@owebeeone/glade-decl` types-only (`grip-core/src/core/share_decl.ts`);
  drift is a compile error. Both workspaces share the same grip-core repo
  at compatible heads.
- **Zones/domains (GDL-039)** — domain→wire `share`, zone→wire `key`,
  `commons | private(self)`; implemented and carried on `ShareDecl`/
  `BindingDecl`.
- **The wire model** — atomic unit `(share, glade_id, key)`, ops keyed with
  `origin`, per-origin hash chains, deterministic `foldValue`/`foldLog`.
  Confirmed real in the running node.
- **The product theses** — mock→real as pure provider swap (C1), the
  doc/environ/instance scope model, one-MCP-door agent delegation (C3).
  Nothing on the glade side contradicts them; they're just unserved.

## What neither side has (the true gap list)

1. **No gryth providers exist as code.** Workspace directory: RegistryApi
   trait exists in the node, `dir.workspaces` was served as an ordinary
   binding in R3, but the directory build (GDL-032) is pending. pty, file
   tree/content, chat transcript, VM management: nothing.
2. **Class-2 streams unexercised.** `Terminal.Output` needs the `stream`
   shape (live/replay split); no end-to-end path exists on either side.
3. **Environ-scope replication unbuilt.** The vision's roaming desktop
   document (C2) has no implementation surface anywhere; gryth's desktop
   isn't even serialized/persisted locally yet (GrithReview-55 slices 1/5).
4. **Attribution stubbed on both sides.** `PrincipalRef` is a bare type in
   gryth; glade's identity/capability enforcement is a problem statement
   (`GladeGrythSecurityModelAnalysisPrompt.md`), grants unenforced (M-LIMP).
5. **Chat scope contradiction (gryth-internal).** Vision says doc-scope
   append-log spine; the plugin holds per-tab instance transcripts and the
   demo proposal says a share decl there "would be wrong." Must be
   adjudicated before chat can ever go real.
6. **Known glial holes** relevant to any consumer: GAP-11 (locally-minted
   offline ops never ship on later attach) and GAP-10 tail (per-tab IDB
   stores never evicted; retention TTL unenforced).

## Decisions needed (agenda for the coordination agent)

1. **Ratify the seam.** GDL-035/036/037/038 are open-proposed and GDL-039
   is "pending ratify" on the live DecisionLog; everything gryth builds on
   sits on unratified decisions. (The DecisionLog note that GDL-039's
   grip-core changes are "uncommitted" looks stale — they are on grip-core
   main as `1cf6ab3`/`8965577`.)
2. **Bless the attachment story.** Confirm: matcher flip selects the
   provider; a GlialTap-backed tap *is* the real provider. Explicitly
   retire `GrythDemoProposal.md` §4 (binder plan) — supersede or rewrite
   the demo proposal against glial mounts.
3. **Consumability of glial.** Is `GlialTap`/`FillSpec` stable enough for
   an external workspace to consume now? How does `gryth-wz` take the
   dependency — add `glial` as a gwz member (as was done for
   `glade-decl-ts`) or wait for a published package? What's the pin/version
   story while it churns?
4. **Pick the first real surface.** Candidates in cheapness order:
   `Doc.WorkspaceName` (a `value` — trivially servable by the node today),
   then `Workspace.List` ↔ `dir.workspaces` (value/log; aligns with the
   pending GDL-032 directory build). Define its `BindingDecl` (glade id,
   shape, authority, domain, zone, retention) as the concrete deliverable.
5. **grazel ownership.** Who writes `grazel-app.glade`, where does it live,
   and who builds the grazel provider sessions — gryth side or glade side?
6. **Fix the doc plumbing.** Re-point `gryth-wz/dev-docs` symlinks
   (`glade/`, `grip-share/`, `DecisionLog.md`, `StackMap.md`) from frozen
   `glial-dev` to `glade-wz`, or replace with dated snapshots. Today gryth
   cannot see glade-side doc drift as a diff.
7. **Adjudicate the open contract questions** that block gryth surfaces:
   chat scope (above), selection sharing (GDL-030 / GSA-OQ-003),
   advertisement record format (GSA-OQ-001/002/004), multi-workspace mount
   topology (one socket per workspace?).

## Gryth-side sequence (valid regardless of coordination outcome)

1. Commit the pending workspace housekeeping (pnpm migration, grip-react
   tsconfig fix, glade-decl-ts member addition).
2. **Phase 3 matcher flips against a fake second binding** — needs zero
   glade. This locks the consumer-side contract and is exactly what makes
   step 4 a drop-in later.
3. Desktop schema + persistence slices (GrithReview-55 slices 1/5) — the
   environ story needs a serializable desktop document no matter what the
   substrate is.
4. First glial-backed provider behind the flip, on whichever surface the
   coordination picks in agenda item 4.
