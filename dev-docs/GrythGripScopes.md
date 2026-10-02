# Gryth grip scopes

Date: 2026-10-02
Status: **design, ruled** (owner, 2026-10-02: "yes to all", §3.4) — the scope of
every grip the desk defines, for the owner's two-node settings demo. The inventory
below preserves its original source snapshot; implementation status is in §4.
Source: `gryth-wz` main at `afa5766`, with gryth-ui as locked there; read only.
Method: every `defineGrip` call was read with its comment, and a tap only where it
decides what holds a value. No build, test or server was run.
Plans cited: "the appearance plan" is glade-wz's
`dev-docs/glial/GlialAppearanceSettingsPlan.md`, and "the cross-node plan" is
glade's `dev-docs/GladeCrossNodeWritesPlan.md`.

The ask (owner, 2026-10-02): two nodes, each with a browser, with synchronized
settings. Zoom, font size, theme and background image are the user's settings,
shared across all of the user's sessions. One session opened in two browsers also
shares its grid locations. Every grip gets a scope.

**In short.**

- Add a **session** scope between environ and instance. Environ keeps the user's
  settings, a session is one desk, and instance stays one page.
- The appearance is already environ on the Gyld desk (the Glial appearance plan,
  merged 2026-09-28). The grid geometry, 12 grips, moves from a per-browser
  localStorage blob to session. A page names its session with `?session=`, the way
  it names its user with `?principal=`.
- All 214 `defineGrip` calls have a scope: 18 doc, 11 environ, 36 session,
  6 promotable, 101 instance, 21 derived and 21 wiring. 45 grips change scope.

## 1. The scope model, as the desk needs it

### 1.1 The scopes

The first four come from `GrythVision.md` ("The google-docs model") and from
`GripLabGripAndTapInventory.md`, which adds the promotable middle.

- **doc**: the shared artifact on a share: workspace state, terminals, chat, VMs
  and Gyld output. It is durable, multi-writer and attributed, and everyone who
  resolves the same key sees the same value.
- **environ**: the user's own durable state. It is the same in every session,
  browser and node of that user, and the user can delegate it to an agent. In this
  design it holds the user's settings and nothing else.
- **instance**: one running page: a drag, a hover, a menu, the canvas size, a
  scroll position, a draft, the connection. It is never stored and never
  replicated.
- **promotable**: a tag, not a store. It marks a focus, a selection or a chosen
  set that may be promoted to doc scope as a shared focus (follow or presenter
  mode; GDL-030, open). Until it is promoted, it lives in session scope.
- **session** (new, §1.3): one desk: its windows and pane grid, its current
  desktop, its sidebar and what each window shows. It is durable, and it is the
  same in every page that opens that session.

Two kinds of grip hold no state of their own, so the inventory gives them a class
instead of a scope:

- **derived**: a class 2 cache or class 3 conversion that a tap computes from
  other grips. Nothing stores it, and it has its inputs' scope.
- **wiring**: plugin identities, intents, the registry and handle objects. Every
  page rebuilds them at load, and they act at the scope of what they write.

As built, environ is per user and per app. The appearance surface is
`<entry>.appearance` (appearance plan §1), so the full desktop and the Gyld desk
would keep separate settings.

### 1.2 The owner's case

GrythVision's environ holds both halves of the case: "the user's desktop: window
layout, focus, selections, filters", replicated across all of the user's instances
(`GrythVision.md:21-24`). The owner's case splits it in two:

- the settings follow the user into every session, on every node;
- the grid follows a session: two browsers on one session share it, and two
  sessions of one user do not.

The code does neither yet. The appearance roams on the Gyld desk only. The layout
stays in each browser's localStorage (`gryth.desk.layout.v1.<entry>`,
`packages/desktop/src/layoutStorageTap.ts:56`). Two tabs of one browser read it at
load and overwrite each other later, with no live sync, and two browsers never
share it.

### 1.3 Recommendation: a session scope between environ and instance

| Scope | Shared by | Holds |
| --- | --- | --- |
| environ | every page of one user | the settings |
| session | every page that opens one session | one desk |
| instance | one page | view state in motion |

One session in two browsers is one mirrored desk. Two sessions are two independent
desks over the same settings and the same documents. This answers GrythVision's
open question, "mirrored vs independent" (`GrythVision.md:179-182`), without the
per-instance current-desktop pointer that question expected: a reader who wants an
independent view opens another session. It is not the "session scope" that
GrythVision retired, which joined environ and instance (`:28-30`). The new scope is
durable and replicated, like environ, and it is keyed by a session as well as a user.

**Per-tab grips.** The desk builds a tab's grips from the tab's record once, when
the tab first appears (`packages/desktop/src/tabContexts.ts:67`). The record holds
the tool, the link params and the wire, and it is part of `Desktop.Windows`, so it
is session. A per-tab grip that the link carries is session through the record.
The other per-tab grips (camera, selection, hover, menus, drafts) are instance, so
two browsers on one session may pan the same picture differently.

Alternatives:

1. **Each session its own environ, with settings as doc** on an account share
   (`account:<p>`). Settings are private and single-writer, which doc does not
   describe. The account share also has no claim, so it would fork across nodes
   (appearance plan §1).
2. **One environ value that holds every session.** The four scopes stay, but a
   move in any session rewrites the user's whole value, and sessions contend under
   last-writer-wins.
3. **Per-browser desks, plus an act that sends a desk to another browser.** This is
   today's storage. It shares nothing live, so it does not meet the case.
4. **Promotion**: one browser follows the other's layout. Promotion is one-way and
   is meant for sharing with other people, but the owner's two browsers are equal
   writers of one desk.

### 1.4 The scope spec

**Today no page names a session.** Five inputs decide what a page reads and
writes:

| Input | From | Selects |
| --- | --- | --- |
| entry | the page loaded: the full desktop or the Gyld entry | the blob key `gryth.desk.layout.v1.<entry>` and the surface `<entry>.appearance` |
| principal | `?principal=` (or `?user=`); else `principal` in grazel's `/bootstrap.json` (gyld-ui `--principal`, default `owner`); else the tab's own id (`packages/glade/src/identity.ts:13-26`) | the appearance key `self:<principal>`. Only the first two roam (`DeskIdentity.roams`) |
| browser profile | the localStorage of the page's host | the blob: one desk per profile and entry |
| tab origin | sessionStorage `glade-origin`, held by a Web Lock; a duplicated tab mints its own (`identity.ts:84-136`) | the op chain only, not a scope |
| node | `node_ws` in `/bootstrap.json` | the node the page talks to |

`?principal=` and `?user=` are the only URL parameters gryth-ui reads
(`packages/glade/src/bootstrap-util.ts:30-33`). `Glade.Identity` and
`Settings.Appearance.Follows` report the principal, but neither one selects
anything.

**Proposed: the scope spec is (principal, entry, session).**

- `?session=<name>` names the session. The page reads it with `?principal=`,
  before it composes.
- Without the parameter, the page opens its browser's own session for that entry
  and principal. That session is minted once and kept in localStorage, so a browser
  reopens its own desk, as the blob lets it do today.
- A second browser that opens the same session as the same user mirrors the desk.
  A second tab of one browser mirrors it by default.
- The settings window names the session beside the user and gives a link that
  opens the session elsewhere.
- A page with no named user keeps today's browser-only desk, because it has no zone
  to hold a session in.

### 1.5 Where each scope lives

| Scope | Today | Proposed |
| --- | --- | --- |
| doc | glial mounts of glade shares (the chat, gwz and gyld logs); mock atoms for the rest | unchanged |
| environ | the `gyld.appearance` value on `ws-razel`, private zone, key `self:<principal>` (Gyld desk, named user); the blob otherwise | `<entry>.appearance` on every entry. The full desktop is the appearance plan's question 9, ruled for later |
| session | nothing; the desk lives in the per-browser blob | one `<entry>.desk` value per user and session, in the private zone on `ws-razel` (question 3) |
| instance | page memory and tab contexts | unchanged |

## 2. The inventory

**Counting.** The listed files hold 232 occurrences of `defineGrip`: 214 calls, 15
imports, and 3 more in `packages/plugin-api/src/runtime.ts` and `index.ts` (the
definition, a comment and the re-export). Each call is counted once. Two calls in
`chat/src/groups.ts` are templates that make one grip for each pre-declared group
(`general` and `dev`), so a running page holds 216 grips.

**Rows.** "`X` (+ `.Tap`)" is a value grip and its handle grip, `X.Tap`. They share
a producer and a scope, and they count as 2. The "Held by" column uses these terms:

- **identity**: a plugin's registry key;
- **atom**: an atom tap in page memory;
- **atom/tab**: an atom with one copy per tab, held in that tab's context;
- **record**: an atom/tab that its window writes back into its tab record, so the
  blob keeps it;
- **blob**: the interim desk document in localStorage;
- **roamed**: the glial appearance value in the user's private zone;
- **share**: a glial mount of a glade doc share;
- **mock**: an atom standing in for a doc provider;
- **derived**: computed by a tap from other grips.

"Today" is the scope a comment or the code states, or "—" where none is stated. A
**bold** proposal differs from today's scope.

### 2.1 Plugin identity: 9

| Grips | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Desktop.BuiltinTools.Plugin`, `Settings.Plugin`, `Terminals.Plugin`, `Chat.Plugin`, `Workspace.Plugin`, `Code.Plugin` (in `code/src/index.ts`), `Gwz.Plugin`, `Vm.Plugin`, `Gyld.Plugin` (9) | the plugin's registry key | identity | — | wiring | rebuilt at every load |

### 2.2 plugin-api (`grips.ts`, `registry.ts`): 10

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Doc.WorkspaceName` | the workspace's name | a glial value on a local binder with no glade destination, seeded `mock-workspace` (`src/taps.ts`) | doc | doc | never leaves the page today |
| `Desktop.OpenTool`, `Desktop.RetargetTab`, `Desktop.OpenWired`, `Desktop.SetTabSource`, `Desktop.OpenWiredPair`, `Desktop.PinTab` (6) | the shell's open, retarget, wire and pin intents | atom holding a function | — | wiring | they write `Desktop.Windows`, so they act at session scope |
| `Desktop.TabLinks` | every tab's tool and link params | derived from `Desktop.Windows` | — | derived | session, through its input |
| `Plugins.Registry` (+ `.Tap`) (2) | the plugin map of component factories | atom, never stored | — | wiring | |

### 2.3 The app (`src/taps.ts`): 1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Doc.WorkspaceName.GlialControl` | the write controller of `Doc.WorkspaceName` | the glial tap's controller | — | doc | the handle of a doc value |

### 2.4 desktop (`grips.desktop.ts`): 41, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Desktop.Windows` (+ `.Tap`) (2) | frames, geometry, z-order, foundations (the pane grid), docks, and tab records with their link params and wires | atom + blob | environ: "persists and roams" | **session** | the grid locations. Stored per browser and never roamed |
| `Desktop.GridMemory` (+ `.Tap`) (2) | each desktop's stashed grid, which the lock restores | atom + blob | environ | **session** | |
| `Desktop.FoundationPreset` (+ `.Tap`) (2) | the pane preset a lock opens | atom set by the target, + blob | environ | **session** | |
| `Desktop.Current` (+ `.Tap`) (2) | which virtual desktop is shown | atom + blob | environ, flagged as per-instance | **session** | question 4 |
| `Desktop.SidebarOpen`, `Desktop.SidebarWidth` (+ `.Tap` each) (4) | the sidebar's state and width | atom + blob | environ | **session** | geometry; not in the owner's list of settings |
| `Desktop.FocusedWindow` (+ `.Tap`) (2) | the focused frame | atom, not stored | environ, flagged as per-instance | **instance** | each page keeps its own focus |
| `Desktop.Theme`, `Desktop.UiZoom`, `Desktop.FontScale`, `Desktop.Wallpaper`, `Desktop.WallpaperThemed` (+ `.Tap` each) (10) | the appearance | on the Gyld desk with a named user: roamed, projected from `Settings.Appearance.Value`. Otherwise: the settings plugin's atoms + blob | environ | environ | the full desktop still uses the blob (`src/compose.tsx:15-18`) |
| `Desktop.ResetLayout` | forget the stored desk | atom holding a function | — | wiring | under session scope it resets the session's desk |
| `Desktop.WindowDrag`, `Desktop.TickerHover`, `Desktop.TickerBleed`, `Desktop.WindowMenu`, `Desktop.AreaMenu`, `Desktop.CanvasSize`, `Desktop.DeskSlide`, `Desktop.Overview` (+ `.Tap` each) (16) | a drag, the ticker hover and picker, menus, the canvas size, the desktop slide, the overview | atom | instance | instance | |

### 2.5 glade (`runtime.ts`): 5

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Glade.Identity` | the principal, the tab's origin, and whether the principal roams | the grip's default, fixed at load | — | instance | the principal half of the scope spec |
| `Glade.Status` (+ `.Tap`) (2) | connecting, live or offline | atom set by `startGlade` | — | instance | |
| `Glade.Node` (+ `.Tap`) (2) | the node URL from `/bootstrap.json` | atom | — | instance | |

### 2.6 settings (`grips.ts`): 2, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Settings.Appearance.Value` | the appearance document `{v, theme, zoom, fontScale, wallpaper, wallpaperThemed}` | roamed: `gyld.appearance` on `ws-razel`, private zone, key `self:<principal>`, stored in IndexedDB. Gyld desk with a named user only | environ, "and roamed" | environ | |
| `Settings.Appearance.Follows` | the principal this page's appearance follows | derived from the identity by the projection tap | instance | instance | reports the scope spec, selects nothing |

### 2.7 terminals (`grips.ts`): 2, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Terminals.Sessions` (+ `.Tap`) (2) | PTY sessions and who started them | mock | doc | doc | a PTY session, not a desk session |

### 2.8 chat (`grips.ts`, `groups.ts`): 4 calls, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Chat.Group` (+ `.Tap`) (2) | the group a chat window shows | atom/tab, seeded `general` | instance, per tab | **session** | the window's destination. A reload loses it today |
| `Chat.${g.id}.Lines` (+ `.Tap`) (2 calls, 4 grips) | a group's lines and its post handle | share: the glial commons log `chat.msgs`, keyed per group | doc, "shared/global" | doc | |

### 2.9 workspace (`grips.ts`): 6, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Workspace.List` (+ `.Tap`) (2) | the workspaces, with their repos and edges | mock | doc | doc | |
| `Workspace.Viewer.Selected` (+ `.Tap`) (2) | the workspace a viewer shows | atom/tab | instance, per viewer | **session** | the window's destination. A reload loses it today |
| `Workspace.Viewer.GraphNodes` | the graph simulation's nodes | the tab's simulation tap | per viewer | instance | presentation only |
| `Workspace.Viewer.GraphEngine` | the simulation engine, which takes gestures | the tab's simulation tap | per viewer | wiring | |

### 2.10 code (`grips.ts`): 3, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Code.WTA` (+ `.Tap`) (2) | an explorer's workspace, path and ref: what a wired viewer follows | atom/tab, seeded from the link | instance, per source | **promotable** | the "current file". Changes are not written back to the link |
| `Code.SourceAccent` | a source tab's colour | derived from the tab id | per source | derived | the same in every page of a session |

### 2.11 gwz (`grips.ts`): 8, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Gwz.Verb`, `Gwz.Result`, `Gwz.RunId` (+ `.Tap` each) (6) | the verb picker, the last answer, the streaming run | atom, page-wide | "shared across windows" | instance | |
| `Gwz.Stream` (+ `.Tap`) (2) | a run's output records | share: the glial log `gwz.output`, keyed by run id | "shared across windows" | doc | the comment groups it with the page atoms |

### 2.12 vm (`grips.ts`): 18, identity in §2.1

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Vm.Machines`, `Vm.Bases` (+ `.Tap` each), `Vm.Providers`, `Vm.Profiles` (6) | vm_manager's machines and bases; the host's providers and profiles | mock | doc | doc | |
| `Vm.Form.Name`, `Vm.Form.Profile`, `Vm.Form.Provider`, `Vm.Form.Base`, `Vm.Form.BaseName`, `Vm.Form.Error` (+ `.Tap` each) (12) | a manager window's create-form draft | atom/tab | instance | instance | |

### 2.13 gyld (`grips.ts`): 105, identity in §2.1

The comments call every per-tab atom "INSTANCE scope, one set per tab". The rows
below separate what a tab record carries (session) from what it does not
(instance).

| Grip | Holds | Held by | Today | Proposed | Note |
| --- | --- | --- | --- | --- | --- |
| `Gyld.Dest.Stream`, `Gyld.Dest.Perspective`, `Gyld.Dest.Preview` (+ `.Tap` each) (6) | a browser's stream, perspective and previewed question | record: every pick writes them back (`browser/destination.ts:46-54`) | instance | **session** | derived in diff panes and in detail windows that follow the focus |
| `Gyld.Tab.Legend` (+ `.Tap`) (2) | the legend overlay: shrunk, expanded or help | record | per window | **session** | |
| `Gyld.Dest.Ref` (+ `.Tap`) (2) | the record a window is on | atom/tab, seeded from the link's `ref` or `focus` | instance | **session** | a browser's own pick is camera-side and is not written back |
| `Gyld.Dest.Left`, `Gyld.Dest.Right`, `Gyld.Dest.Run`, `Gyld.Dest.Proposal` (+ `.Tap` each) (8) | a diff's two streams; a compare's run and proposal | atom/tab, seeded from the link | instance | **session** | later changes are not written back |
| `Gyld.Tab.Ask.Conversation` (+ `.Tap`) (2) | the conversation an ask window is on, and its record | atom/tab, seeded from the link | instance | **session** | later changes are not written back |
| `Gyld.Set` (+ `.Tap`) (2) | the bundle roots the desk reads | atom, not stored; the landing tap re-adds the glade root at each boot | environ, "persisted and roamed"; doc-promotable | **promotable** | nothing stores it; question 7 |
| `Gyld.Focus` (+ `.Tap`) (2) | the cross-window "what are you looking at" | atom at the plugin root | environ, share-promotable | **promotable** | session until promoted |
| `Gyld.Ops.Result`, `Gyld.Ops.RunId`, `Gyld.Ask.Conversation` (+ `.Tap` each) (6) | the last supplier answer; the run and the conversation the logs follow | atom (`live.ts`) | instance, desk-wide | instance | |
| `Gyld.Ops.Stream` | a run's output | share: the glial log `gyld.output` on `ws-razel`, keyed by run id | instance | **doc** | it folds a doc log, though its comment calls it instance |
| `Gyld.Ask.Stream` | a conversation's reply | share: the glial log `gyld.ask` on `ws-razel`, keyed by conversation | — | doc | |
| `Gyld.Ops`, `Gyld.Store.Reload` (2) | the operations handle; the reload handle | handle objects | — | wiring | |
| `Gyld.Streams`, `Gyld.Store.Status`, `Gyld.Bundle`, `Gyld.DecideNow`, `Gyld.Validation`, `Gyld.Lens`, `Gyld.Sources`, `Gyld.Diff`, `Gyld.Run`, `Gyld.Comparison` (10) | Gyld output read from the set's roots for each destination, and the read status | derived: `GyldStoreTap` | class 2 | derived | caches doc truth |
| `Gyld.Records`, `Gyld.Record`, `Gyld.Preview` (3) | conversions for each destination; a preview layout | derived | class 3 | derived | |
| `Gyld.Lens.Palette` | the drawing colours | derived from `Desktop.Theme` | class 3 | derived | environ, through its input |
| `Gyld.Landing`, `Gyld.Ops.Status`, `Gyld.Node` (3) | what an empty desk lands on; copies of `Glade.Status` and `Glade.Node` | derived | — | derived | instance, through its inputs |
| `Gyld.Tab.Id`, `Gyld.Dest.Side` (2) | the owning tab's id; a compare pane's side | atom/tab, fixed when created | — | derived | fixed by the tab record or the pane |
| `Gyld.Tab.Camera`, `Gyld.Tab.Camera.Drag`, `Gyld.Tab.Selection`, `Gyld.Tab.Hover`, `Gyld.Tab.Dimmed`, `Gyld.Tab.Flash`, `Gyld.Tab.Card.Size`, `Gyld.Tab.Search`, `Gyld.Tab.Follow`, `Gyld.Tab.Menu`, `Gyld.Tab.Diff.Slot` (+ `.Tap` each) (22) | a picture's camera, pan, selection, hover, dimmed classes, flash and card size; the search text, the follow toggle, the node menu, and a diff's record in hand | atom/tab | instance | instance | |
| `Gyld.Tab.Ask.Draft`, `Gyld.Tab.Ask.AtEnd`, `Gyld.Tab.Ask.Answer`, `Gyld.Tab.Ask.Cites` (+ `.Tap` each) (8) | an ask window's draft, scroll position, last answer and open citations | atom/tab | instance | instance | |
| `Gyld.Tab.Run.Draft`, `Gyld.Tab.Picker.Url`, `Gyld.Tab.Picker.Error`, `Gyld.Streams.Draft`, `Gyld.Streams.Export`, `Gyld.Tab.Draft.Answer`, `Gyld.Tab.Draft.Ask`, `Gyld.Tab.Draft.Taken`, `Gyld.Tab.Export.Answer`, `Gyld.Tab.Export.Ask` (+ `.Tap` each) (20) | drafts and exported command text | atom/tab | instance | instance | |
| `Gyld.Tab.Draft.Took`, `Gyld.Tab.Picked` (2) | what a decide window applied; what a browser picked for itself | a per-tab tap's own state | — | instance | |

**Total: 214 calls, all accounted for (216 grips at run time).**

## 3. Summary

### 3.1 Counts per proposed scope

| Scope | Calls |
| --- | --- |
| doc | 18 (20 grips at run time) |
| environ | 11 |
| session | 36 |
| promotable | 6 |
| instance | 101 |
| derived | 21 |
| wiring | 21 |
| **total** | **214** |

### 3.2 What changes scope: 45 grips

- **environ → session, 12:** `Desktop.Windows`, `Desktop.GridMemory`,
  `Desktop.FoundationPreset`, `Desktop.Current`, `Desktop.SidebarOpen` and
  `Desktop.SidebarWidth`, each with its handle.
- **environ → instance, 2:** `Desktop.FocusedWindow` (+ `.Tap`). Nothing stores it
  today, so only its label moves.
- **instance → session, 24:** the link grips `Gyld.Dest.Stream`, `.Perspective`,
  `.Preview`, `.Ref`, `.Left`, `.Right`, `.Run` and `.Proposal`, plus
  `Gyld.Tab.Legend`, `Gyld.Tab.Ask.Conversation`, `Chat.Group` and
  `Workspace.Viewer.Selected`, each with its handle.
- **to promotable, 6:** `Gyld.Set` (environ, never stored), `Gyld.Focus` (environ,
  promotable) and `Code.WTA` (instance, per tab), each with its handle.
- **instance → doc, 1:** `Gyld.Ops.Stream`, which only its comment calls instance.

Some holders must change even where the scope does not: the full desktop's
appearance must move from the blob to the roamed value, and the doc mocks
(`Doc.WorkspaceName`, `Terminals.Sessions`, `Workspace.List` and the `Vm.*` lists)
must give way to real providers.

### 3.3 What the settings demo needs

1. **The appearance set, 11 grips:** `Desktop.Theme`, `Desktop.UiZoom`,
   `Desktop.FontScale`, `Desktop.Wallpaper` and `Desktop.WallpaperThemed`, their
   handles, and `Settings.Appearance.Value`. These are already environ. On the Gyld
   desk with a named user they are one roamed value (appearance plan Phases 1 to 3,
   merged into the owner's gryth-wz on 2026-09-28), so no grip changes. The full
   desktop still keeps them in its blob, and across two nodes the value is only as
   shared as `ws-razel` is (question 8).
2. **The grid geometry, 12 grips** (§3.2, first line): these must move from
   environ in a per-browser blob to session, as one value per user and session
   (questions 2 and 3).
3. **The 24 link grips** ride in the tab records inside `Desktop.Windows`, so they
   move with it. The second browser shows their changes live only once the desk
   re-applies a changed record (question 5).
4. **New:** a session in the scope spec (question 2).

### 3.4 Questions for the owner

1. **Add the session scope?** Recommend yes. Environ narrows to the user's
   settings, the desk moves to session, and instance is unchanged. Record the
   change in GrythVision's tiers and against GDL-030. §1.3 lists the alternatives.
2. **How a page names its session.** Recommend `?session=<name>`, read beside
   `?principal=`. Without it, the page uses a session minted once per browser
   profile, entry and user and kept in localStorage, so each browser keeps its own
   desk as it does today, and two tabs of one browser mirror it live (today they
   overwrite each other's). The settings window names the session and links to it.
   The alternative is gyld-ui `--session`, served in `/bootstrap.json`, which makes
   every browser on one node share one desk.
3. **Where a session lives.** Recommend one glial `value`, `<entry>.desk`, on
   `ws-razel` in the user's private zone, keyed `self:<principal>/<session>`. The
   `/` is a character no principal name may hold (grazel allows only
   `A-Z a-z 0-9 . _ -`), so the appearance plan's Step 4.3 (not built) can read
   the principal up to it. The whole document is last-writer-wins and debounced as
   the blob is (300 ms), and the blob seeds it once, as Step 2.4 seeded the
   appearance. Split it per window only when a two-browser check loses a move
   (`grips.desktop.ts:16-17`). The alternative is one glade id per session, which no
   app file can declare.
4. **The current desktop and focus.** Recommend `Desktop.Current` as session, so
   two browsers on one session switch desktops together, and
   `Desktop.FocusedWindow` as instance, so each page keeps its own keyboard focus.
   A reveal still raises the frame in both browsers, through the z-order. The
   alternative is a current desktop per page: GrythVision's per-instance pointer.
5. **Tab records changed by the other browser.** Recommend that a tab's link grips
   follow its record: when the record changes elsewhere, the desk re-applies it to
   the live tab context, which today is seeded once (`tabContexts.ts:67`). Chat's
   group, the viewer's workspace and the explorer's WTA join their links, so a
   reload keeps them too. The camera, selection, hover and drafts stay per page.
   The alternative is to mirror every per-tab grip, drafts included.
6. **The reveal cue inside the desk document.** `WindowRecord.attention` is
   instance state stored in `Desktop.Windows`. A restore already drops it
   (`layoutDocument.ts:203-262`), but a live session value would play the cue in
   both browsers, and both sweeps would write the clear. Recommend moving it to an
   instance grip before the desk becomes a session value.
7. **`Gyld.Set`.** Recommend session, promotable to doc for a set the team shares,
   because it is what this desk reads. Today nothing stores it, and the landing tap
   re-adds the glade root at each boot. The alternative is environ: one set for
   every desk of the user.
8. **The two-node demo.** Recommend running it on the Gyld desk, whose appearance
   is already environ, and leaving the full desktop to the appearance plan's
   question 9, as ruled. Run both nodes on one machine (owner, 2026-10-02): NAT
   and firewall traversal is iroh's job, so a second machine proves nothing
   glade-specific, and the two-machine run is a later integration test.

   Each gyld-ui node loads `gyld-app.glade` and so serves `ws-razel`. The live
   claim with the highest epoch holds it, and the other node forwards its writes
   there (the cross-node plan, built). A node mints its claim at its replica's
   highest epoch plus one (`glade/node/src/claims.rs`), so a node that serves after
   it has the other's claim takes over. The settings are shared only through that
   one holder.

   A glade issue found in review, fixed first (glade `c1a6764`, 2026-10-02): two
   nodes that each serve `ws-razel` before either has the other's claim both
   claim epoch 1, and glade's two folds of the claims broke that tie apart. The
   routing fold (`node/src/mesh/route.rs`, `who_serves`) kept the first claim in
   origin order, and the registry's (`node/src/registry.rs`), which
   `Server::serves` and the assembly's host port use, kept the last. Both now rank
   by `registry::rank_claims`: the higher epoch, then the lower node id.

**Ruled, owner, 2026-10-02 ("yes to all"):** every recommendation above, 1 to 8.
The tie-break of question 8 was fixed first, in glade `c1a6764`.

## 4. Implementation, 2026-10-02

The accepted session/appearance split is implemented in the `appearance` lane.
See [GladeSettingsSessionPlan.md](GladeSettingsSessionPlan.md) for the contract,
boundaries, regression tests and verification. Named Gyld sessions use one
`gyld.desk` value; live link fields bridge tab records and their existing grip
contexts in both directions. Gyld roots/focus and Code WTA remain doc-promotable.
The reveal cue is page-local. Promotion UX and the full desktop's roaming remain
deferred. The original inventory's “Today” column describes the recorded source
commit, rather than the implemented tree.
