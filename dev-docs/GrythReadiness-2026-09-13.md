# Gryth readiness for a new external plugin family

Date: 2026-09-13
Status: **assessment + phased plan** - input for the decision to host a Gyld
decision-graph browser and decision-taker plugin family in `gryth-ui`.
Scope: this document assesses `gryth-wz` only. It does **not** design the Gyld
plugins; a separate effort owns their design, and nothing here should be read as
constraining their internals beyond the shell contract they must sit on.
Method: read-only survey of `/Users/owebeeone/limbo/gryth-wz` plus verification
runs of `pnpm test`, `pnpm lint`, `pnpm build`, `pnpm why` and a standalone
`tsc --noEmit` probe, all executed 2026-09-13. No source file was modified.

## Evidence legend

Every claim below is tagged.

- **(V)** Verified by running a command in this session. The command is named.
- **(R)** Read from the tree (source, manifest, lockfile, or git history) and
  inferred. High confidence, but no execution proved it.
- **(U)** Unverified. Stated as a hypothesis with the command that would settle
  it. Nothing in the plan depends on a **(U)** claim being true.

## TL;DR

- **The tree is red, and it is red for two independent reasons.** (V) `pnpm test`
  gives 64 passing tests but 2 test *files* that fail to load; `pnpm lint` fails
  with exactly one error; `pnpm build` fails at `tsc -b` with exactly one error.
  The two causes are (1) a one-character corruption, a stray `0` prefixed to
  line 1 of `packages/plugins/chat/src/groups.test.ts`, and (2) a genuinely
  missing dependency, `@owebeeone/taut-shape`, which the `glial` member now
  requires. Fixing the stray `0` alone will **not** green the build: it only
  unblocks `tsc -b` far enough to reach the second failure. Both must land.
- **`@owebeeone/taut-shape` is published on npm.** (V) `pnpm view` reports
  versions `0.9.0`, `0.9.1`, `0.9.2` with `latest: 0.9.2`, which satisfies
  glial's `^0.9.1`. The three resolution options in the brief are therefore not
  a hard trilemma: a plain `pnpm install` in `gryth-ui` should fetch it from the
  registry, and that is also the option glial's own history already chose.
- **The workspace lockfile is stale against the `glial` member, and a workspace
  link pins nothing.** (R) The `../glial` importer block in
  `gryth-ui/pnpm-lock.yaml` (lines 101 to 119) records only
  `@owebeeone/glade-decl` as a dependency; `taut-shape` appears nowhere in the
  lockfile at all. `glial` has declared `@owebeeone/taut-shape` since
  `ab9922c` (2026-08-22). `node_modules` was last installed 2026-08-22 19:34.
  This is the whole of the breakage: glial moved and gryth-ui did not reinstall.
- **The biggest architectural finding is not in the brief: `gryth-ui` resolves
  two different copies of the Glial kernel at once.** (V) `pnpm why
  @owebeeone/glial-runtime` shows the root app bound to `link:../glial` (the
  `gryth-wz` member, `da06989`) while every plugin package and `@owebeeone/glade-chat`
  is bound to `file:../../glade-wz/glial` (a *different* checkout of the *same*
  repo, at `0dfe4b9`). Both call themselves `@owebeeone/glial-runtime` version
  `0.0.0`. The `overrides` block in `pnpm-workspace.yaml` forces a grip
  singleton and has **no** entry for `glial-runtime`. This MUST be adjudicated
  before a new plugin family is added, because a new plugin is exactly the thing
  that would pick a side and then fail to see the other side's state.
- **`dev-docs/GlialWorkspaceNameCutover.md` (untracked) makes two claims that are
  false today.** It says the lock "records the clean Glial revision used by this
  cutover" (a `link:` workspace entry records no revision) and that
  `pnpm-workspace.yaml` "makes Glial and Grip resolve as workspace singletons"
  (grip yes, Glial no, per the `pnpm why` above). Its acceptance evidence was
  true when written against glial `eefc9c8`; two later glial commits moved under
  it.
- **The uncommitted work is two coherent changesets plus one accident**, and they
  are entangled only through `pnpm-lock.yaml`. Splitting them is cheap in intent
  and awkward in practice for exactly that reason. Recommendations are in
  section 2; the call is Gianni's.

## 1. Tree state and hygiene

### 1.1 Where every member actually sits

(V) `git -C <member> log -1` and `git -C <member> status --porcelain`, all
2026-09-13.

| Member | Branch | Head | Date | Subject | Working tree |
|---|---|---|---|---|---|
| `gryth-wz` (root) | `main` | `7316086` | 2026-07-11 | Status report - what is what with glial integration being next. | **dirty** (7 entries, 4 staged) |
| `gryth-ui` | **`glp-0006-p1s4-gryth-panels`** | `bffbdd0` | 2026-07-12 | GLP-0006 P1.S4: live glade panels (chat + gwz) served by grazel | **dirty** (8 modified, 3 untracked) |
| `glial` | `main` | `da06989` | 2026-08-29 | Add canonical SWMR assembly | clean |
| `grip-core` | `main` | `97ff6c2` | 2026-07-10 | lock file push | clean |
| `grip-react` | `main` | `c13b8a7` | 2026-07-11 | Transitioned from gryth-dev and moved to pnpm | clean |
| `glade-decl-ts` | `main` | `7059c1b` | 2026-08-29 | Render SWMR declaration contract | clean |

Two corrections to the briefing facts, both material:

1. **`gryth-ui` is not on `main`.** It is on `glp-0006-p1s4-gryth-panels`. All
   the uncommitted work therefore sits on a feature branch whose head is from
   2026-07-12, while the Glial cutover work in the working tree is from
   2026-08-22. Whether that branch is still the right home for an August cutover
   is a question for Gianni, not an assumption this plan should make.
2. **The root repo is also dirty**, and partly *staged*. (V) `git -C
   /Users/owebeeone/limbo/gryth-wz status --porcelain`:

   ```
    M .claude/launch.json
   A  .claude/settings.json
   A  AGENTS.md
   M  AGENTS_GWZ.md
   M  gwz.conf/gwz.lock.yml
   M  gwz.conf/gwz.yml
   A  gwz.conf/markers/conf-integrity.yml
   ```

   `AGENTS_GWZ.md` and everything under `gwz.conf/` are gwz-managed. Per
   `AGENTS_GWZ.md`, structural changes MUST go through the `gwz` CLI and hand
   edits to `gwz.conf/` are refused by gwz on the next structural command. The
   staged `gwz.conf` changes plus the new `markers/conf-integrity.yml` look like
   the output of a `gwz` command rather than a hand edit, since a digest marker
   was written alongside them **(R)**. This SHOULD be confirmed before any
   further `gwz` operation, with `gwz status` from the workspace root.

### 1.2 The two blockers, precisely

**Blocker A: a stray `0`.** (V) `git -C gryth-ui diff -- packages/plugins/chat/src/groups.test.ts`
is a single-line change:

```
-import { describe, it, expect } from 'vitest';
+0import { describe, it, expect } from 'vitest';
```

This is not a partial edit or a rebase artifact with meaning; it is a
keystroke. It is also, by itself, the *entire* content of the lint and build
failure:

- (V) `pnpm lint` output: `1:1 error Parsing error: An identifier or keyword
  cannot immediately follow a numeric literal`, then `1 problem (1 error, 0
  warnings)`. One error, no others. Everything else in the repo lints clean.
- (V) `pnpm build` output: `packages/plugins/chat/src/groups.test.ts(1,2): error
  TS1351`, then exit. `tsc -b` stops here, so the build never reaches `vite
  build` and never reaches Blocker B.
- (V) `pnpm test`: the file fails to transform (`esbuild` "Syntax error
  \"i\"").

**Blocker B: `@owebeeone/taut-shape` is not installed.** (V) `pnpm test`:

```
FAIL  src/seam.test.ts [ src/seam.test.ts ]
Error: Cannot find package '@owebeeone/taut-shape' imported from
  '/Users/owebeeone/limbo/gryth-wz/glial/src/swmr.ts'
```

The causal chain, fully pinned down:

- (R) `glial/package.json` declares `"@owebeeone/taut-shape": "^0.9.1"` as a
  production dependency.
- (V) `ls /Users/owebeeone/limbo/gryth-wz/glial/node_modules/@owebeeone/` shows
  only `glade-decl` and `grip-core`. No `taut-shape`. Confirms the brief.
- (R) `grep -n taut-shape gryth-ui/pnpm-lock.yaml` returns nothing. The lockfile
  has no `taut-shape` entry and its `../glial` importer block lists only
  `@owebeeone/glade-decl`.
- (R) `node_modules` and `node_modules/.pnpm` are both dated 2026-08-22 19:34,
  matching the cutover doc's own date of 2026-08-22.
- (R) `git -C glial log -S'taut-shape' -- package.json` shows the specifier
  history: `ab9922c` (2026-08-22) introduced it as `file:../taut-shape-ts`;
  `03578db` (2026-08-26, "Pin Glial to released Taut shape package") changed it
  to `^0.9.1`. At `file:../taut-shape-ts` it would have resolved, from the
  `gryth-wz/glial` member, to `gryth-wz/taut-shape-ts`, which (V) does not
  exist. So the specifier that is *installable* under `gryth-wz` is the current
  registry one, and it postdates the last install.
- (R) `glial/src/index.ts` re-exports from `./swmr.ts`,
  `./provider_status_atom.ts`, `./live_metrics_stream.ts` and
  `./collaborative_text.ts`, which are (V, by grep for `from "@owebeeone/taut-shape"`)
  exactly the four modules that import taut-shape. The barrel entry point is
  therefore unconditionally load-bearing on taut-shape. This matters for
  planning: the failure **cannot** be side-stepped by avoiding SWMR or by
  importing a narrower subpath through `.`; any consumer of
  `@owebeeone/glial-runtime` needs taut-shape resolvable.

Blocker B also breaks type checking, not just the test runner. (V) A standalone
probe that writes nothing, `pnpm exec tsc --noEmit --skipLibCheck --target es2022
--module preserve --moduleResolution bundler --jsx react-jsx --strict
src/taps.ts`, reports:

```
../glial/src/collaborative_text.ts(11,8): error TS2307: Cannot find module
  '@owebeeone/taut-shape' or its corresponding type declarations.
```

Caveat, stated so nobody chases it: that probe also emitted many `TS5097`
"import path can only end with a '.ts' extension" errors. Those are **artifacts
of the ad-hoc flags**, not real findings; (V) the project's own
`tsconfig.app.json` and `tsconfig.node.json` both set
`"allowImportingTsExtensions": true`. Only the `TS2307` is a real resolution
failure, because bare-specifier resolution does not depend on that flag.

### 1.3 Minimal ordered path to a green tree

Blocker A must be fixed before Blocker B is *observable* in the build, but the
two are independent and can be fixed in either order. The ordering below is
chosen so that each step's verification command gives a clean signal.

**Step H1. Revert the stray `0`.** The change carries no information, so the
correct edit is a revert of that one file, not a hand fix:

```sh
git -C /Users/owebeeone/limbo/gryth-wz/gryth-ui checkout -- \
  packages/plugins/chat/src/groups.test.ts
```

Verify:

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui && pnpm lint
```

Expected: exits 0 with no output. (R) This is the only lint error in the tree, so
lint SHOULD go green on this step alone.

**Step H2. Resolve `@owebeeone/taut-shape`.** Four options, not three. Trade-offs:

| Option | Edit required | Pros | Cons |
|---|---|---|---|
| **(a) Plain reinstall from the registry** (recommended) | none to any manifest; run `pnpm install` in `gryth-ui` | Matches the specifier glial already committed (`03578db` deliberately pinned to the *released* package). No manifest churn. taut-shape 0.9.2 is (V) published and satisfies `^0.9.1`. Refreshes the whole stale lock in one action. | Rewrites `pnpm-lock.yaml`, which is already part of an uncommitted changeset, so it entangles hygiene with the commit adjudication in section 2. Needs network. |
| **(b) Install inside the `glial` member only** | none; `pnpm install` in `gryth-wz/glial` | Narrow blast radius; leaves `gryth-ui`'s lock untouched. | (R) Fights the topology: `glial` is a *workspace package* of `gryth-ui` via `pnpm-workspace.yaml`, so its deps belong in gryth-ui's lock. A member-local `node_modules` may satisfy the runtime while leaving `gryth-ui`'s lock still stale, so CI stays red. Creates a second lockfile in the member **(U)**, check with `ls gryth-wz/glial/pnpm-lock.yaml`. |
| **(c) Add `taut-shape-ts` as a pnpm workspace entry** | `pnpm-workspace.yaml` gains a path; requires a `taut-shape-ts` checkout under `gryth-wz` (there is (V) none today) plus a `gwz repo add` | Source-level debugging across the shape boundary; consistent with how grip and glial are already handled. | Directly contradicts the standing decision: `GlialWorkspaceNameCutover.md` states "Gryth does not link to the uncommitted Glial checkout in `taut-dev`" and glial's `03578db` moved *away* from a `file:` shape link. Adds a member to the workspace for a leaf dependency. Most work, least benefit. |
| **(d) Pin an exact published version** | `glial/package.json` `^0.9.1` becomes `0.9.2` | Deterministic. | Changes a *clean* member's manifest to fix a *dirty* consumer's install. Wrong repo for the edit. Only worth it if 0.9.x float turns out to be unstable. |

Recommendation: **(a)**, and note that it subsumes the lock refresh that
Blocker B needs anyway. Verify:

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui && pnpm install \
  && pnpm test && pnpm lint && pnpm build
```

Expected after H1 and H2: `pnpm test` reports 10 passed test files and 66+ tests
(the two currently-failing files contribute their own tests once they load);
`pnpm lint` exits 0; `pnpm build` completes `tsc -b` and then `vite build`.
Marked **(U)**: nothing in this session proved the *post-fix* state, because
proving it requires the install. The residual risk is that `src/seam.test.ts`
fails on assertion rather than on load, which would be a real cutover bug rather
than a hygiene problem, and which this plan deliberately does not pre-judge.

**Step H3. Decide the `--frozen-lockfile` question.** (R) The untracked
`gryth-ui/.github/workflows/taut-shape-consumer.yml` runs `pnpm install
--frozen-lockfile` and then `pnpm test`, `pnpm run build`, `pnpm run lint`.
Because the committed lock's `../glial` importer omits `taut-shape` while
glial's manifest requires it, that job would fail at the install step with an
outdated-lockfile error **(U)**; the exact failure mode is not verified here,
since running it would rewrite the lock. The lock refresh in H2 is what makes
that workflow viable. Verify, once the lock is refreshed and *before* trusting
CI:

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui && pnpm install --frozen-lockfile
```

Expected: exits 0 and reports the lockfile is up to date.

Note that this workflow also (R) checks out eight repositories and symlinks
`workspace/glial` into `glade-wz/glial`, which means CI deliberately makes the
two glial paths the *same* tree. Local development does not. That divergence is
the subject of section 3.

## 2. Adjudicating the uncommitted `gryth-ui` work

(V) `git -C gryth-ui diff --stat` plus per-file diffs. The eight modified files
and three untracked paths sort into three groups.

### Group 1: the Glial workspace-name cutover (coherent, self-documented)

- `pnpm-workspace.yaml`: adds `- ../glial` to `packages:`.
- `package.json`: adds `"@owebeeone/glial-runtime": "workspace:*"`.
- `src/taps.ts` (+51 / -22): replaces `createAtomValueTap(WORKSPACE_NAME, {
  initial: 'mock-workspace' })` with a Glial-backed `glialTap`, introducing
  `GrythSurfaces = defineManifest({ workspaceName: { id: 'gryth.workspace.name',
  shape: 'value', share: 'gryth-local' } })`, a provider-side
  `WORKSPACE_NAME_CONTROL` grip typed `GlialTapController<string>`, a module-level
  `workspaceNameBinder = new GlialBinder(undefined, 'gryth-ui')`, and a
  once-only seed of `'mock-workspace'` through the Glial write/fold path.
- `src/seam.test.ts` (+12 / -6): rewrites the first test to drive
  `WORKSPACE_NAME_CONTROL.set('glial-workspace')` and assert the unchanged
  `WORKSPACE_NAME` consumer observes it.
- `dev-docs/GlialWorkspaceNameCutover.md` (untracked): the design note, dated
  2026-08-22, with a "Contract preserved" section that is properly normative
  (consumers MUST keep reading `WORKSPACE_NAME`, MUST NOT import Glial).
- `.github/workflows/taut-shape-consumer.yml` (untracked): the CI job that
  proves gryth-ui as a taut-shape consumer. (R) An identically-named file exists
  at `gryth-wz/glial/.github/workflows/taut-shape-consumer.yml`, so this is the
  consumer-side half of a deliberate pair, not a stray.

Assessment: this is one commit's worth of work with a doc and a CI gate
attached, and it belongs together. Two caveats Gianni SHOULD weigh:

1. The doc's "Dependency boundary" section is now **factually wrong** in two
   places, as set out in the TL;DR and section 3. If the group is committed
   as-is, the false claims enter the record. Recommend amending that section in
   the same commit.
2. The doc's "Acceptance evidence" paragraph asserts that the full test, build,
   no-React-state and ESLint gates pass. (V) They do not pass today. That
   statement was true against glial `eefc9c8` and was invalidated by glial
   `03578db` and `da06989`. It SHOULD be re-verified (step H2) or re-worded
   before it is committed.

### Group 2: the wyred test mount (coherent, explicitly reversible)

- `package.json`: adds `"@wyredjs/plugin-wyred":
  "link:../../wyred-wz/wyred-ui/packages/plugin-wyred"`.
- `vite.config.ts` (+15 / -4): extends `resolve.dedupe` to
  `['react', 'react-dom', '@grythjs/plugin-api', '@owebeeone/grip-react',
  '@owebeeone/grip-core']`, extends `optimizeDeps.exclude` with
  `'@wyredjs/plugin-wyred'` and `'@wyredjs/artifacts'`, and adds `server.fs.allow:
  ['..', '../../wyred-wz']`.
- `src/plugins/wyred/index.ts` (untracked): a 1-line shim, `import
  '@wyredjs/plugin-wyred';`, with a comment naming
  `wyred-wz/dev-docs/GrythWyredUiDesignPlan.md` section 3.5b step 0.2, stating
  it is reversible by deleting the directory and the manifest line, and
  deferring the permanence question to that plan's step 3.4.

Assessment: also one commit's worth, and notably well-behaved: it is
self-labelled as a verification-time test mount, it names its owning plan and
step, and it states its own exit condition. **This is the closest thing in the
tree to a worked example of hosting an external plugin family, and section 3
treats it as the reference pattern.** The `server.fs.allow` widening is a dev
server concession, not a production change **(R)**.

The judgement call: the wyred mount is owned by a plan in a *different*
workzone (`wyred-wz`). Committing it into `gryth-ui` on a branch named
`glp-0006-p1s4-gryth-panels` puts wyred's step 0.2 into gryth's GLP-0006
history. That may be exactly what was intended, or it may want its own branch.
Recommend, do not decide.

### Group 3: noise and accident

- `packages/plugins/chat/src/groups.test.ts`: the stray `0`. Recommend
  **revert** (step H1). There is no reading under which this is wanted.
- `start-demo.sh` (+3 / -3): purely cosmetic. Three `$PORT` interpolations
  become `${PORT}`; no behaviour change. Recommend folding into whichever group
  is committed first, or dropping it. It belongs to neither changeset.
- `pnpm-lock.yaml` (+494): **this is the entanglement.** (R) It carries both
  Group 1 (`../glial` importer, `@owebeeone/glial-runtime: link:../glial` at
  line 47) and Group 2 (`@wyredjs/plugin-wyred` at lines 54 to 56). It cannot be
  split by hand into two coherent halves.

### Recommended shape of the adjudication

Three honest options, in increasing cost:

1. **Commit Groups 1 and 2 together as one "cross-workspace consumer seam"
   commit**, after H1 reverts the stray `0` and H2 refreshes the lock. Cheapest,
   loses the distinction between the Glial cutover and the wyred mount in the
   history, and is defensible because the lock genuinely fuses them.
2. **Split into two commits, Group 1 then Group 2**, regenerating the lock at
   each step. Costs two `pnpm install` runs and produces a clean history. This
   is the only option that yields a bisectable "Glial cutover" commit.
3. **Commit Group 1, drop Group 2.** Viable if the wyred mount has served its
   verification purpose, since the shim states it is reversible by construction.
   Requires confirming with the `wyred-wz` plan's step 3.4 first.

Recommendation: **option 2** if the Glial cutover is meant to stand as a
reviewable unit (the presence of a dedicated design doc and a dedicated CI
workflow suggests it is), otherwise option 1. Either way H1 lands first and
independently.

**Stop point for Gianni: D1.** Which option, and whether
`glp-0006-p1s4-gryth-panels` is the right branch for August and September work.
Nothing downstream in this plan is blocked by the choice, but everything
downstream is cleaner after it.

## 3. Dependency and workspace topology

### 3.1 What is actually wired

(R) `gryth-ui` mixes four distinct mechanisms at once:

1. **In-repo pnpm workspace packages**: `packages/*` and `packages/plugins/*`,
   depended on as `workspace:*`. These are the `@grythjs/*` scope.
2. **pnpm workspace entries pointing outside the repo**: `../grip-core`,
   `../grip-react`, and (uncommitted) `../glial`. These are `gryth-wz` gwz
   members that are not inside the pnpm root. The manifest comment explains
   why this was chosen over a `file:` override: "A relative `file:` override
   resolves relative to the DEPENDENT package, so it broke once nested packages
   (via glial-runtime's grip-core peer) needed it."
3. **`file:` dependencies reaching into `glade-wz`**: `packages/glade` and
   `packages/plugins/chat` depend on `@owebeeone/glade-chat` and
   `@owebeeone/glial-runtime` at `file:../../../../glade-wz/...`.
4. **A `link:` dependency reaching into `wyred-wz`** (uncommitted):
   `@wyredjs/plugin-wyred` at
   `link:../../wyred-wz/wyred-ui/packages/plugin-wyred`.

Singleton discipline is enforced for grip and only for grip. The `overrides`
block pins `@owebeeone/grip-core` and `@owebeeone/grip-react` to `workspace:*`,
and `vite.config.ts` dedupes them at bundle time.

### 3.2 Is it coherent? Partly, and there is one real break

For grip, yes. (V) `pnpm why @owebeeone/grip-core` shows every path, including
the peer edges from `glial-runtime` and `glade-chat`, terminating at
`link:../grip-core`. One copy. The override does its job.

For Glial, **no**. (V) `pnpm why @owebeeone/glial-runtime` shows two distinct
resolutions live simultaneously:

```
@owebeeone/glial-runtime link:../glial                                    <- the root app
@grythjs/glade link:packages/glade
  └── @owebeeone/glial-runtime file:../../glade-wz/glial(...)             <- every plugin
@grythjs/plugin-chat link:packages/plugins/chat
  └── @owebeeone/glial-runtime file:../../glade-wz/glial(...)
@grythjs/plugin-gwz link:packages/plugins/gwz
  └── @owebeeone/glial-runtime file:../../glade-wz/glial(...)
```

And those two directories are different code. (V) `git log -1`:

- `/Users/owebeeone/limbo/gryth-wz/glial` is at `da06989` (2026-08-29, "Add
  canonical SWMR assembly").
- `/Users/owebeeone/limbo/glade-wz/glial` is at `0dfe4b9` (2026-08-29, "Add
  collaborative text CRDT demo").

(V) Both have remote `git@github.com:owebeeone/glial-runtime.git`; both declare
`"name": "@owebeeone/glial-runtime"`, `"version": "0.0.0"`, and
`"@owebeeone/taut-shape": "^0.9.1"`. So they are two divergent checkouts of one
repo, indistinguishable to pnpm by name or version, and pnpm keeps them both.

Why this matters for the task at hand, stated plainly: Glial is described in its
own package description as "the client-side kernel ... refcounted binding
instances". The uncommitted `src/taps.ts` constructs `new GlialBinder(undefined,
'gryth-ui')` from the `../glial` copy, while every glade-backed panel in the app
goes through the `glade-wz/glial` copy. Two kernels means two binder and
instance registries, two store engines, and no shared state between the app's
own Glial surfaces and the plugins' Glial surfaces **(R)**. A new plugin family
that wants to share Glial-mediated state with the shell would pick one copy and
silently not see the other. This is the same class of defect the grip override
exists to prevent.

The `.github` CI workflow papers over this: (R) it symlinks a single glial
checkout into both locations, so CI would test a configuration that local
development does not have. That makes CI green a weaker signal than it looks.

Verify the break, and later verify any fix:

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui && pnpm why @owebeeone/glial-runtime
```

Expected today: two distinct resolutions. Expected after a fix: one.

Two further coherence notes:

- (R) `tsconfig.app.json` carries a long comment recording that
  `noUnusedLocals`, `noUnusedParameters` and `erasableSyntaxOnly` were **turned
  off program-wide** for GLP-0006 P1.S4, because the app now deep-typechecks the
  leniently-configured *source* of the glade-wz packages and "There is no
  declaration boundary short of a composite-references restructure". It names
  the exit condition: "Restore when the glade packages ship built .d.ts
  (publish-or-member call)." Any new source-level cross-workspace dependency
  inherits this cost, and widens it.
- (R) `gryth-wz/dev-docs` still contains four **symlinks into
  `/Users/owebeeone/limbo/glial-dev`**: `DecisionLog.md`, `StackMap.md`,
  `glade/`, `grip-share/`. (V) They all resolve, so they look healthy, but (V)
  `glial-dev`'s last commit is `cb5113d` (2026-06-17). `GrythGladeSeamAssessment.md`
  already flagged this in July ("Gryth has been reading a dead workspace").
  Three months on, the symlinks are still there and still silently serving
  2026-06 content. This is a documentation trap for any new contributor and
  SHOULD be repointed or removed.

### 3.3 Hosting a new plugin family that lives in `gyld-wz`

Four candidate mechanisms. The comparison is about *this* shell's constraints,
not general packaging taste.

| Mechanism | Concrete edits in `gryth-ui` | Pros | Cons |
|---|---|---|---|
| **(i) The wyred pattern: `link:` dep + a one-line `src/plugins/<name>/index.ts` shim** | `package.json`: add `"@gyldjs/plugin-<name>": "link:../../gyld-wz/<pkg-path>"`. `vite.config.ts`: add the plugin's shared singletons to `resolve.dedupe`, add the package to `optimizeDeps.exclude`, add `'../../gyld-wz'` to `server.fs.allow`. New file `src/plugins/<name>/index.ts` containing the side-effect import. Import that from the composition root. | Proven in this tree *today*. Smallest diff. Explicitly reversible (delete directory plus one manifest line). Keeps the plugin's source in its own workzone and its own git history. Lets the plugin family iterate without a publish cycle. | `link:` is a bare symlink, so pnpm records no integrity and no version. Drags the plugin's own transitive deps into the app's resolution, where they can duplicate singletons unless dedupe is extended for *each* one. Inherits the `tsconfig` strictness relaxation of section 3.2. Not releasable. |
| **(ii) pnpm workspace entry, as grip and glial have** | `pnpm-workspace.yaml`: add `- ../../gyld-wz/<pkg>` (or a `gyld` member under `gryth-wz`). Probably an `overrides` entry to force the singleton. Likely a `gwz repo add` if the code should be a `gryth-wz` member. | Strongest singleton control: `workspace:*` plus `overrides` resolves from any depth, which is precisely the problem the manifest comment says `file:` could not solve. Correct mechanism for anything that MUST be a singleton. | Heaviest structural change; touches gwz-managed state, so it MUST go through the `gwz` CLI. Couples two workzones' checkouts. Reaching `../../` out of the workzone for a workspace entry is unprecedented in this tree; every existing entry is a sibling `../`. |
| **(iii) Published package** | `package.json`: add `"@gyldjs/plugin-<name>": "^0.1.0"`. Nothing else. | Cleanest boundary, real versioning and integrity, ships `.d.ts` so it would *not* inherit the strictness relaxation, and it is the only option compatible with a gryth release. It is also the direction glial's own `03578db` chose for taut-shape. | Requires a publish pipeline for a family that does not exist yet, and a publish for every iteration. Wrong tool while the plugin design is still moving. |
| **(iv) `file:` dep, as `packages/glade` uses for glade-wz** | `packages/<host>/package.json`: add `"@gyldjs/plugin-<name>": "file:../../../../gyld-wz/<pkg>"`. | Consistent with the existing glade seam. pnpm records it as a directory resolution rather than a bare link. | (R) This is the mechanism that produced the two-glial split, because a relative `file:` resolves relative to the *dependent* package. The manifest comment in `pnpm-workspace.yaml` explicitly records that it "broke". Recommend against. |

**Recommendation, in two stages.** Start with **(i), the wyred pattern**, for
the development phase: it is the only option already demonstrated to work in
this tree, it is the cheapest to reverse, and it does not touch gwz-managed
state. Plan the migration to **(iii), a published package**, as the release
gate, which is what `GlialWorkspaceNameCutover.md` already asserts as policy
("A Gryth release MUST depend on a released Glial semver"). Use **(ii)** only
for any artefact that must be a process-wide singleton shared with the shell.
Avoid **(iv)**.

One condition attaches to that recommendation, and it is not optional: whatever
mechanism is chosen, the new family's shared runtime singletons MUST be added to
both the pnpm `overrides` block and `vite.config.ts`'s `resolve.dedupe` at the
same time as the dependency. Section 3.2 is what happens when they are not.
Verify after each such addition:

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui \
  && pnpm why @owebeeone/grip-core \
  && pnpm why @owebeeone/glial-runtime \
  && pnpm why <each new shared package>
```

Expected: exactly one resolution per shared package.

**Stop point for Gianni: D2.** The hosting mechanism, and the related question
of whether the two-glial split is fixed before or after the new family is
mounted. Fixing it first is strongly preferable if the Gyld plugins will touch
Glial-mediated state at all.

### 3.4 The mount mechanism is already a drop-in convention

This is the good news in the whole assessment, and it was not in the brief.

(V) `src/plugins/index.ts` is the composition root and its first statement is:

```ts
const modules = import.meta.glob('./*/index.ts', { eager: true });
void modules;
```

Every `src/plugins/<name>/index.ts` is therefore loaded eagerly at bootstrap
with **no edit to any shared file**. (V) `git status --porcelain --
src/plugins/index.ts` is clean even though the wyred mount was added, which
proves the glob picked it up: `src/plugins/wyred/index.ts` is (V) referenced
from nowhere in `src/`, `packages/` or `vite.config.ts`, and it still runs. The
wyred shim's claim that it is reversible by "delete this directory and the
package.json dep line" is accurate.

So mounting a new plugin family costs one new directory plus one dependency
line. No registry edit, no shared-file merge conflict, and it is parallel-safe:
two agents adding two plugin directories do not touch the same file.

Two caveats a planner MUST account for:

- The glob is eager and unconditional, so creating `src/plugins/gyld/` changes
  the boot behaviour of **every** existing test that boots the app, not just new
  ones. Any regression it causes will look like a failure in an unrelated
  suite. Verify with a full `pnpm test` immediately after creating the
  directory, before adding any plugin logic.
- `import.meta.glob` is a Vite feature. It works under vitest because vitest
  uses Vite, but it means the mount list is not visible to plain `tsc`, and a
  plugin that is mounted only by the glob will not appear in any import graph a
  reader greps for.

### 3.5 The premise gap: there is no TypeScript package in gyld-wz

(V) `find /Users/owebeeone/limbo/gyld-wz -name "*.ts" -o -name "*.tsx"` returns
nothing, and `find ... -name package.json` returns nothing. `gyld-wz` is a gwz
workspace with a single member, `gyld/`, whose head is (V) `47b4bb5`
(2026-09-08, "Extract standalone Gyld application from workzone"). It is a
**Python** project: `pyproject.toml`, `src/`, `tests/`,
`scripts/check_architecture.py`, plus `.mypy_cache` and `.ruff_cache`. (R) Its
`README.md` states the application lives in the private `owebeeone/gyld`
repository registered as member `mem_gyld`.

This materially changes the answer to "how should gryth host a plugin family
that lives in `gyld-wz`". The wyred pattern works because
`wyred-wz/wyred-ui/packages/plugin-wyred` **exists** as a TypeScript workspace
package to link to. `gyld-wz` has no analogue, so option (i) in section 3.3 has
an unstated prerequisite: somebody must first scaffold a TS package. That is
real work and it belongs in the plan as its own step rather than being smuggled
into "add a dependency line".

There are two ways to supply that package, and the choice is a genuine fork:

| | **(A) In-repo first: `gryth-ui/packages/plugins/gyld/`** | **(B) External from the start: a new TS member in `gyld-wz`** |
|---|---|---|
| Mechanism needed | None. `workspace:*` like the seven existing plugins. | A `gwz repo add` or `gwz repo create` in `gyld-wz`, plus the wyred-pattern `link:` wiring in `gryth-ui`. |
| Gates it inherits | All of them: eslint, `tsc -b`, and (V, section 3.4) the no-React-state scan, since `packages/plugins/*/src` is a scan root. | (V) None of gryth's gates automatically. Section 3.6 applies. |
| Strictness cost | None beyond the existing relaxation. | (R) Inherits the `tsconfig.app.json` source-seam relaxation, and widens it. |
| Git home | gryth's history. | Gyld's history, which is arguably where a Gyld plugin belongs. |
| Cost to reverse | Extracting a package later is mechanical but is a real commit. | Cheap to delete, per section 3.4. |
| Release path | Already releasable with gryth. | Needs the publish pipeline of section 3.3 option (iii). |

Recommendation: **start with (A)** and treat extraction to (B) as a later,
optional step. The reasoning is that (A) requires *no new hosting mechanism at
all*, it keeps the plugin inside every quality gate while its shape is still
moving, and it removes the two-workzone coupling from the critical path of a
plugin family that does not exist yet. Choosing (B) first buys a cleaner git
home at the cost of inventing the hosting story and the CI story before there is
any plugin code to justify them. If the Gyld plugins are expected to ship
independently of gryth, (B) becomes correct, but that is a product decision.

One consequence of either choice is worth naming, because it is the largest
unknown in this whole area and it is nobody's assigned topic: **the Gyld
application is Python, and the plugin is browser TypeScript.** No data path
between them exists today. Whether that path is a glade supplier, an HTTP or WS
API, or a generated static artefact is out of scope here, but it MUST be settled
before the plugin family can show real data, and it is likely to be the longest
lead-time item. Flagged as an open dependency, not a recommendation.

### 3.6 A gate that an external plugin would escape

(V) `scripts/no-react-state.test.mjs` computes its scan roots as exactly three
things: `<root>/src`, every `packages/*/src`, and every
`packages/plugins/*/src`. An external plugin body living in `gyld-wz` sits under
none of them, so **the no-React-state rule would not be enforced against it**.
The wyred mount demonstrates the shape of the hole: (R) the shim at
`src/plugins/wyred/index.ts` is scanned, but the plugin body it imports from
`wyred-wz` is not.

This is not an argument against external hosting, but it is a condition on it.
Whichever mechanism is chosen in D2, the plugin family's own repository MUST run
the equivalent gate itself, or `no-react-state.test.mjs` MUST be taught the new
root. The cheapest honest option is the former, since the script is
self-contained and the external repo can vendor it. Verify:

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui && node scripts/no-react-state.test.mjs
```

Expected: `OK: no unapproved React hooks found in src/ (7 banned, 0 approvals).`
(V) This passes today; it runs as the first half of `pnpm test` and is not
implicated in either blocker.

## 4. Graph rendering readiness

This section reports state only. It deliberately does **not** choose a graph
library or a layout algorithm; that belongs to the Gyld plugin design effort.
What it does do is state precisely what the shell provides, what it does not,
and what it would constrain.

### 4.1 What exists today

(R) All graph code in the repository lives in one plugin package and nowhere
else, at `/Users/owebeeone/limbo/gryth-wz/gryth-ui/packages/plugins/workspace/src/`:

| File | LOC | Role |
|---|---|---|
| `graphEngine.ts` | 263 | `GraphEngine` force simulation plus `GraphSimTap` grip wiring |
| `WorkspaceGraph.tsx` | 155 | Pure SVG view |
| `WorkspaceViewer.tsx` | 57 | The `ToolDef.windowComponent` |
| `grips.ts` | 31 | `GRAPH_NODES`, `GRAPH_ENGINE` and friends |
| `types.ts` | 41 | `RepoInfo`, `DepEdge`, `GraphRenderNode` |
| `mock.ts` | 58 | The only data source, a static fixture |

The renderer is a hand-rolled force simulation: springs plus inverse-square
repulsion plus gravity toward the viewBox centre, axis-of-least-penetration
overlap separation, wall bounce and friction, driven by `requestAnimationFrame`
and self-stopping below a settle threshold. It draws repo cards, confirming the
briefing note: each node is an SVG group wrapping a `foreignObject` HTML card
showing repo name, branch, SHA, a dirty pill, ahead/behind badges and, when
expanded, a clickable changed-files list.

Interaction support is partial. Drag, hover and a single-node "pin" selection
all work and are routed as method calls on the engine. **There is no camera**:
(R) `WorkspaceGraph.tsx` uses a fixed `viewBox="0 0 1080 680"` with
`preserveAspectRatio="xMidYMid meet"`, which is scale-to-fit only. No zoom, no
pan, no wheel or pinch handling.

The model is hard-wired to the repo domain, not generic. `GraphEngine.setInput`
takes `RepoInfo[]` and `DepEdge[]` directly, and repo-specific fields are baked
into the render node type:

```ts
export interface DepEdge { source: string; target: string; }
```

`DepEdge` has no id, no kind and no style channel, and `GraphRenderNode` carries
`branch`, `head`, `ahead`, `behind`, `dirty` and `changedFiles` as first-class
fields rather than an opaque payload.

### 4.2 What the docs specify and the code does not have

(R) `GraphRendererRequirements.md` and `GraphRendererFeatures.md` (both July)
frame the work as *replacing* this engine behind a seam, with named
non-functional requirements: NF3 "swappable behind an explicit seam" (a
`GraphRenderer` contract plus a generic node/edge model with a domain
`payload`), and NF4 "layout is a strategy, not hard-wired" (force as one
implementation among others, selected at runtime).

None of it exists. (V) A grep across `packages/` and `src/` for
`GraphRenderer|RendererContract|LayoutStrategy|LayoutEngine|renderer seam`
returns zero matches. (V) `grep -cE "dagre|elkjs|cytoscape|reactflow|@xyflow|sigma|vis-network|d3-force"
pnpm-lock.yaml` returns `0`, so no graph or layout library is present even
transitively. The briefing note is correct and can be stated more strongly: the
renderer seam is not partially built or stalled mid-flight, it is unstarted, and
(R) the graph source files were last touched in `ebbd9cd` (2026-06-13) while the
three renderer docs were added four weeks later in `5ee4d06` (2026-07-10).
Nothing has touched the graph code since.

(R) `GrazelGraphProtocol.md` is likewise marked "proposal v3" and is
unimplemented on the UI side: a grep for its own vocabulary (`get_graph`,
`expand_node`, `expand_edge`, `StatusRollup` and so on) returns zero matches.
The workspace graph's data comes from the `mock.ts` fixture through
`createAtomValueTap(WORKSPACE_LIST, ...)`.

### 4.3 What a new graph plugin can and cannot reuse

It cannot reuse the existing graph code. (V) `packages/plugins/workspace/package.json`
declares `"exports": { ".": "./src/index.ts" }`, and that index re-exports only
`WORKSPACE_PLUGIN`, `WORKSPACE_LIST`, `WORKSPACE_LIST_TAP` and the
`WorkspaceRecord` type. `GraphEngine`, `GraphSimTap`, `WorkspaceGraph`,
`GraphRenderNode` and `DepEdge` are **not** exported. There is no
`packages/graph`. So the force engine is private by construction, and a new
graph browser would have to bring its own renderer.

That is the correct conclusion for planning purposes, and it is convenient: it
means **the shell needs no graph work at all** for a new graph plugin to exist.
What the shell already provides, and what a new plugin SHOULD use rather than
reinvent, is the surrounding machinery (all in
`packages/plugin-api/src/registry.ts` unless noted):

- `ToolDef` with `windowComponent`, `label`, `defaultSize`, `role` and
  `menuTitle`, registered by side-effect import through `addEntry`.
- `ToolDef.tabTaps`, the per-tab state seeding mechanism. This is the load-bearing
  one for a graph viewer, because it is how each open window gets its own
  independent engine instance.
- The cross-plugin navigation grips the workspace graph itself already uses to
  turn a node click into an action elsewhere: `DESKTOP_OPEN_WIRED_PAIR`,
  `ToolLink`, `DESKTOP_TAB_LINKS`, `DESKTOP_RETARGET_TAB`. A decision-graph
  browser that opens a decision detail view on node click SHOULD reuse these
  rather than invent a navigation channel.

### 4.4 The one hard constraint the shell imposes

The no-React-state rule is not advisory and it is mechanically enforced (section
3.4). For a graph viewer, which is the most state-heavy thing one can build,
this dictates the architecture outright. The existing engine is the sanctioned
worked example, and (R) `CodingRules.md` names it as such for the "animation and
timers" case.

The mechanism, which a new graph plugin MUST mirror: simulation and interaction
state live as **private fields on a plain class instance**, not in React. The
class is wrapped by a `BaseTap` subclass that owns the animation loop and starts
or stops it on grip-consumer connect and disconnect, publishing immutable
snapshots into a grip:

```ts
export class GraphSimTap extends BaseTap {
  readonly graph = new GraphEngine();
  private publishNodes = (n: GraphRenderNode[]) => {
    this.publish(new Map([[GRAPH_NODES as never, n as never]]));
  };
  onConnect(dest, grip) { super.onConnect(dest, grip); this.graph.attach(this.publishNodes); }
  ...
}
```

The tap is seeded per tab via `tabTaps` (verified in
`packages/plugins/workspace/src/index.ts`), the view component reads the
snapshot with `useGrip(GRAPH_NODES)`, and every gesture is a direct method call
on the engine handle obtained from `useGrip(GRAPH_ENGINE)`. No component holds
local state of any kind.

The practical consequence for the Gyld effort, which is worth stating because it
is easy to get wrong: **most third-party React graph libraries hold their own
state in hooks internally.** That does not by itself violate the rule as
enforced, since (V) the scan only covers first-party `src` trees and a
`node_modules` library is never scanned. But a library whose *integration
surface* requires the plugin to call `useState` or `useEffect` in its own code
would violate it. This is a real selection constraint on the other agent's
library choice, and it is the shell's only strong opinion on the matter.

### 4.5 What the shell MUST and MUST NOT provide

- The shell MUST NOT be asked to grow a generic graph renderer or a layout
  strategy seam as a precondition. NF3 and NF4 are a workspace-plugin
  refactor, not a hosting requirement, and coupling the Gyld family to them
  would serialise two independent efforts.
- The shell MUST keep `tabTaps` working as the per-window state seam, because a
  graph viewer with no per-tab engine is not buildable under the hooks ban.
- The shell SHOULD expose nothing new for graphs. If two graph consumers later
  want to share a renderer, that is the moment to extract `packages/graph` and
  build the NF3 seam properly, with two real clients to design against rather
  than one.
- Open question, not for this document: whether the Gyld browser should consume
  `GrazelGraphProtocol.md` (a proposal with no implementation on either side) or
  define its own data path. Flagged because picking the unimplemented protocol
  would silently add glade-side work to the critical path.

## 5. Plugin contract status

This is the section that most directly answers "what can a new plugin rely on
today". The summary is better than expected: **the registration and composition
machinery is real and tested, and the gaps are all in cosmetic and
provider-swap territory**, not in the load-bearing seam.

### 5.1 Implemented, with code to point at

All paths below are under
`/Users/owebeeone/limbo/gryth-wz/gryth-ui/packages/`.

| Contract element | Status | Where |
|---|---|---|
| Plugin registry tap | **Implemented** | `plugin-api/src/registry.ts`: `PLUGIN_REGISTRY` (`'Plugins.Registry'`), `PLUGIN_REGISTRY_TAP`, built with `createAtomValueTap`; `addEntry`/`removeEntry` copy-on-write the map; `pluginFrom`/`allTools` read it. Registered once from `src/taps.ts`. Covered by `plugin-api/src/registry.test.ts`, including the headless path that drives the registry through the advertised tap handle. |
| Tab contexts and `tabTaps` | **Implemented** | `desktop/src/tabContexts.ts`: `tabContextFor(grok, tabId, def, params)` creates a chrome-held context per tab, runs `def.tabTaps?.(...)` once, and a reaper subscribed to `DESKTOP_WINDOWS` unregisters the taps when the tab leaves the document. Mounted in `desktop/src/Window.tsx` inside a `GripProvider`. Covered by `desktop/src/tabContexts.test.ts`. Used for real per-instance state by 4 of the 7 plugins. |
| Links | **Implemented** | `plugin-api/src/registry.ts`: `ToolLink { toolId; params? }` and `DESKTOP_OPEN_TOOL`; tap in `desktop/src/taps.desktop.ts` (`OpenToolTap`). Wired into the real launcher, with the comment that the launcher opens through the same intent plugins and agents use. `DESKTOP_TAB_LINKS` and `DESKTOP_RETARGET_TAB` are also implemented and consumed. |
| Wired pairs | **Implemented**, and more completely than the docs suggest | Four intents: `DESKTOP_OPEN_WIRED`, `DESKTOP_OPEN_WIRED_PAIR`, `DESKTOP_PIN_TAB`, plus the wire itself as `TabRecord.source`. The wire is a genuine context-graph parent edge (`home.addParent(...)` in `tabContexts.ts`), not a copied value. The contract doc's worked example, a workspace-graph file click opening an explorer and a viewer together, is really built, in `plugins/workspace/src/WorkspaceViewer.tsx`. |
| Open `ToolId` | **Implemented** | `ToolId = string`, fully open. Resolved through `resolveTool`/`allTools` in `desktop/src/facets.ts`, with a `MissingToolFacet` placeholder for unknown ids. |

Notably, wired pairs are **not** in `PluginMigration.md`'s phase plan at all (R):
they landed in `5ee4d06` (2026-07-10), after the migration commits, as a
`GrythPluginContract.md` v2 concept. So the contract doc is ahead of the
migration doc, and the code is ahead of both in this one area.

### 5.2 Not built, and what each costs a new plugin

| Contract element | Status | Consequence for a new plugin |
|---|---|---|
| **Matcher flips** (`withOneOf` / `addBinding`) | **Not built, and superseded** | (V) `grep -rn "addBinding\|withOneOf" packages/ src/` returns **zero matches** in gryth-ui's own code. The primitives do exist in `grip-core`, but gryth uses none of them. The mock-to-real swaps that actually shipped (workspace name, chat, gwz) use **Glial taps** instead, and `GlialWorkspaceNameCutover.md` records that as the forward pattern. A new plugin SHOULD follow the Glial pattern, not `PluginMigration.md` Phase 3. |
| Seeded per-tab grips: `Tool.Id`, `Tool.Params`, `Tool.SourceRef`, `Tab.Focused`, `Tab.DockedArea` | **Not built** | (R) None of these grips exist. `tabId` and `params` arrive as plain React props via `ToolViewProps`. A tool **cannot** learn whether its own tab is focused or which dock area it occupies. A plugin that wants to pause work when hidden has nothing to read. |
| `Tab.Title` and `ToolDef.tabTitle` | **Declared, never consumed** | (V) `grep -rn "tabTitle" packages/ src/` returns exactly two hits, both in `plugin-api/src/registry.ts`: the type field and a comment. There is no `Tab.Title` grip. `TickerStrip.tsx` derives every tab label from the tool's static `label`, so two windows of the same tool show identical titles. A decision-graph browser showing "which graph am I looking at" in the tab **cannot do so today**. |
| `ToolDef.role` | **Declared and populated, never read** | This is the sharpest day-one trap. (V) In all of `packages/desktop/src` and `packages/plugin-api/src`, the string `role` appears exactly **once**, as the optional field declaration carrying the comment `// foundations map roles → areas`. (V) Meanwhile 10 tool definitions across 7 plugins set it. (V) `desktop/src/foundations.ts` assigns areas from a `designate` map keyed on **literal tool ids** (`explorer`, `chat`, `settings`, `terminal`, `diff`, `welcome`) with `fallback: 'stage'`. So a new plugin that sets `role` gets silently ignored and lands in `stage` by fallback. Placing it deliberately requires an edit to `foundations.ts` in the shell. |
| `Tool.View` as a resolved grip | **Not built** | (R) `PluginMigration.md` scopes this to Phase 4 and admits it is out of scope. Today it is a flat registry-map lookup. No impact on a new plugin. |
| Lazy or remote plugin loading | **Not built** | (R) The composition root is the eager glob of section 3.4. An external plugin MUST be source-linked into the same Vite build; it cannot be fetched at runtime. |
| A template plugin | **Does not exist** | (R) `PluginMigration.md` names `src/plugins/welcome` as the example-plugin template; that directory does not exist, and `welcome` was folded into `@grythjs/desktop`'s own builtins. The nearest thing a new author can copy is a real plugin package such as `plugins/workspace`. |
| Publishing | **Not built** | (R) All 10 packages are `"private": true` at `0.1.0`. No changesets config, no publish workflow. This is what makes section 3.3 option (iii) a future rather than a present option. |

### 5.3 Two structural cautions

**The dependency-direction rule is already violated once.** (R)
`PluginMigration.md` Phase -1 states that a plugin package depends only on
`@grythjs/plugin-api` and never on `@grythjs/desktop`. But
`plugins/settings/package.json` depends on **both**, and its `src/taps.ts`
imports `DESKTOP_THEME`, `DESKTOP_ZOOM` and friends straight from
`@grythjs/desktop`. The same document's Phase 1 design causes this, because it
says taps move while grip declarations stay in `grips.desktop.ts`, and never
says how Settings reaches those grips otherwise. `plugins/chat` and
`plugins/gwz` additionally depend on `@grythjs/glade`.

The relevance is direct: a new plugin family SHOULD depend on
`@grythjs/plugin-api` only, and the existence of a counter-example in the tree
means that rule will not enforce itself. If the Gyld plugins need a shell grip
that is declared in `@grythjs/desktop`, they will hit exactly the wall Settings
hit, and the right fix is to move the declaration into `plugin-api`, not to
copy Settings' violation.

**Attribution is a bare type, and a decision *taker* is precisely what needs
it.** (V) `PrincipalRef` is the whole of the attribution surface:

```ts
// Attribution surface for human- and agent-created things (sessions,
// machines, runs): who acted, and on whose behalf.
export interface PrincipalRef {
  principal: string;
  onBehalfOf?: string;
}
```

(V) It is used in exactly one place, `plugins/terminals/src/grips.ts`, as an
`owner` field. (V) A grep of `packages/plugin-api/src` for
`principal|attribut|capabilit|grant` returns only that interface and its
comment. There is no minting, no provider, no enforcement, and no capability or
grant vocabulary anywhere.

`GrythVision.md` commitment C3 is that humans and agents are peers
"distinguished only by capability and attribution", and
`GrythGladeSeamAssessment.md` already recorded in July that attribution is
stubbed on both sides. A **browser** is a read surface and is unaffected. A
**decision taker** writes decisions, and "who decided this, and on whose
behalf" is the first question anyone will ask of such a record. This is flagged
as a gap the Gyld design will run into; it is that effort's call how to handle
it, and this document does not propose a design. But it SHOULD NOT be discovered
late.

### 5.4 What a new plugin can rely on today

In one paragraph, because this is the operative answer. A new plugin can rely
on: registering itself by side-effect import from a dropped-in directory
(section 3.4) or a workspace package; declaring one or more tools with `label`,
`defaultSize`, `windowComponent` and `menuTitle`; receiving `tabId` and `params`
as props inside a per-tab grip context; owning per-window state through
`tabTaps`, including an imperative engine wrapped in a `BaseTap` (section 4.4);
opening other tools through `DESKTOP_OPEN_TOOL`, and opening wired
source-and-sink pairs through `DESKTOP_OPEN_WIRED_PAIR`; reading what is open
through `DESKTOP_TAB_LINKS`; and swapping a mock provider for a live one behind
an unchanged consumer grip using the Glial tap pattern. It cannot rely on
dynamic tab titles, on `role`-based placement, on knowing its own focus or dock
state, on matcher-based provider selection, on lazy loading, or on installing
`@grythjs/plugin-api` from a registry.

## 6. Demo composition

Everything in this section was gathered read-only. Per the brief, `start-demo.sh`
was **not** run, no cargo command was issued, and no binary was built.

### 6.1 The headline: the "gryth demo" starts no backend

(R) `gryth-ui/start-demo.sh` does exactly one thing: `nohup pnpm run dev --port
"$PORT" --strictPort`, with `PORT="${GRYTH_UI_PORT:-5173}"`, a pidfile at
`.demo/run.pid` and a log at `.demo/run.log`. It does not start grazel, the
glade node, or any supplier, and it contains no reference to them.
`stop-demo.sh` kills that one pid. (R) `vite.config.ts` has **no proxy
configuration**, before or after the uncommitted diff, so nothing forwards
`/bootstrap.json` or a WebSocket path from the vite origin to grazel.

So "combined gryth demo and glade demo" is not a matter of merging two runbooks.
Only one of the two halves has a launcher at all, and the gryth half is a bare
dev server.

The reason it works in development anyway is a fallback. (R)
`src/bootstrap.tsx` leads to `packages/glade/src/bootstrap-util.ts`, which
fetches `/bootstrap.json` same-origin, reads `{node_ws?, mode?, name?}`, and
falls back to `DEV_FALLBACK_NODE_WS = 'ws://127.0.0.1:9099'` when the fetch
fails or the field is blank, with a comment tying that constant to "grazel's
default `--node-port 9099`". A vite-served UI therefore reaches a separately
started node by luck of a matching default rather than by configuration.

### 6.2 What grazel is and what it already does

(R) `/Users/owebeeone/limbo/glade-wz/grazel` is a Rust crate with a `grazel`
binary. `--mode local|peer|both` is required and has no default. Other defaults:

| Flag | Default |
|---|---|
| `--name` | `grazel` |
| `--data` | `grazel-data` (relative to cwd) |
| `--http` | `8080` |
| `--node-port` | `9099` (`0` means OS-assigned) |
| `--ui` | `ui` |
| `--app` | `apps/grazel-app.glade` |
| `--node-bin` | `../glade/node/target/debug/glade-node` |
| `--gwz-supplier-bin` | `../glade-gwz/target/debug/glade-gwz` |

It spawns `glade-node` as a subprocess and treats the node's exit as fatal. It
optionally spawns `glade-gwz` as a supplier, which is a loud but non-fatal skip
if the binary is absent, disabled entirely by `--no-suppliers`. It serves `GET
/bootstrap.json` returning `{"node_ws":"ws://127.0.0.1:<port>","mode":"<mode>","name":"<name>"}`
plus static files from `--ui`. Data lands under `--data` as `sys`
(`GLADE_HOME`), `files` (the gwz supplier root), `config`, and
`state/grazel.sqlite3`.

The bootstrap contract therefore **already matches on both sides**, which is the
single most encouraging fact in this section: grazel emits exactly the shape the
UI parses, on the port the UI falls back to.

### 6.3 What is missing or stale

1. **No combined launcher.** This is the actual deliverable gap.
2. **grazel's `--ui` points at a placeholder.** (R) `grazel/ui/index.html` is a
   stub reading, in substance, that gryth-ui arrives at GLP-0006 P1.S4. Nothing
   builds gryth-ui and points `--ui` at its `dist/`. (V) gryth-ui's `dist/` does
   exist but is dated 2026-08-22 19:48, so it predates the uncommitted source
   edits and, more importantly, was built before the tree went red.
3. **Port 9099 is contested.** (R) The older demo at `glade-wz/glade/demo`
   defaults its node to `GLADE_NODE_PORT=9099`, the same default grazel uses, so
   grazel and `glade/demo` MUST NOT be started together without repointing one.
   Their vite ports do not clash (gryth-ui `5173`, `glade/demo` `5175`).
4. **`glade-gwz` shells out to a real `gwz`.** (R) Its `--gwz-bin` defaults to
   the bare name `gwz`, so a `gwz` binary MUST be on `PATH` for the gwz panel to
   do more than attach.
5. **`GrythDemoProposal.md` is superseded and MUST NOT be used as a build
   guide.** (R) It cites paths under `../../glial-dev/` that no longer exist and
   describes a different combined-demo design than the one actually built. This
   is the same `glial-dev` staleness as the `dev-docs` symlinks in section 3.2.
6. **The older `glade/demo` is a prototype, not a peer.** (R)
   `gryth-ui/packages/glade/package.json` describes itself as porting the glade
   demo's `glial.ts`/`glade.ts` into the plugin runtime. So `glade/demo` is the
   thing gryth-ui's seam was ported *from*. Running both in one combined demo
   would be demonstrating an implementation and its own prototype side by side.
   **This SHOULD be adjudicated before scoping the demo**, because it is
   probably not what "combined gryth demo and glade demo" is meant to mean. The
   likelier reading is "one demo where gryth-ui is the UI and the real glade
   node plus suppliers are the backend", which is a much smaller job: it is
   grazel with `--ui` pointed at a real gryth-ui build.

### 6.4 What a combined runbook needs, concretely

- **Processes**: `grazel` (which itself spawns `glade-node` and optionally
  `glade-gwz`), plus either a gryth-ui vite dev server or nothing at all if
  grazel serves a built `dist/`. (R) `glade-chat` is **not** a process; it is
  the TypeScript package `@owebeeone/glade-chat` running in the browser.
- **Binaries**: three independent `cargo build` invocations, because (R) there
  is no Cargo workspace tying them together and each has its own `Cargo.lock`
  and `target/`: `glade-wz/grazel`, `glade-wz/glade/node`, `glade-wz/glade-gwz`.
  Plus `pnpm run build` in gryth-ui, plus a `gwz` on `PATH`.
- **Ports**: grazel HTTP `8080`, glade-node WS `9099`, gryth-ui vite `5173` if
  used.
- **Data dirs**: `<--data>/{sys,files,config,state/grazel.sqlite3}`.
- **App declaration**: (R) `glade/apps/grazel-app.glade` and
  `grazel/apps/grazel-app.glade` are byte-identical, as their own
  dual-maintenance note requires. It declares the app-static bindings, the
  supplier surfaces `gwz.output` and `chat.msgs`/`chat.groups`, the `gwz.ops`
  service with grazel as authority, ACL seeds, and the `ws-razel` / `razel`
  workspace that lines up with `glade-gwz --share ws-razel`.

### 6.5 Risks specific to attempting this now

- **Disk, and it is the binding constraint.** (V) `df -h /System/Volumes/Data`
  reports 17Gi available at 97% capacity, checked at the start and again at the
  end of this session with no change. (R) The three needed `target/` directories
  already total roughly 3.3G and are already built, so incremental rebuilds
  SHOULD be cheap. The plan therefore says: do **not** run `cargo clean`, and do
  not build `glade-discover` (a separate 11-crate workspace with a 2.9G target
  that (R) nothing in this demo references). `iroh 1` in glade-node and
  `rusqlite` with bundled SQLite in grazel's generated store crate are the two
  dependencies most capable of eating the remaining headroom from scratch.
- **Rust builds are out of scope for this assessment.** No cargo command was
  run, so every claim about the Rust side is (R), read from `Cargo.toml`, CLI
  parsing and existing `target/` directories. Whether those binaries still build
  today is **(U)**.
- **Uncommitted glade-wz state means today's tree is not reproducible.** (R)
  `glade-wz` is a gwz workspace with 18 members. Its root is dirty (24 entries),
  and the members `glade-decl-rs` (2 files), `glade` (4 untracked) and
  `glade-discover` (23 entries) are dirty. A fresh checkout would not reproduce
  the current working tree. This SHOULD be settled on the glade side before a
  demo runbook is written against it, or the runbook will encode local state.
- **A second cross-workzone divergence, beyond glial.** (R) `glade-decl-ts` is
  at `7059c1b` in `gryth-wz` and `7e16e32` in `glade-wz`. `grip-core`
  (`97ff6c2`) and `grip-react` (`c13b8a7`) do match. So two of the five shared
  packages are at different revisions in the two workzones, and the two that
  diverge are exactly the two in the Glial and declaration layer. Section 3.2
  covers the glial half; this is the same problem one layer down.
- **Name collision when researching.** (R) `gryth-ui/dev-docs/GrazelGraphProtocol.md`
  uses "razel/grazel" for an older build-graph-viewer concept, and
  `glade-wz/grazel/README.md` explicitly disclaims that earlier usage. These are
  two different things with one name.

## 7. The phased plan

Phases are milestones and land in order. Steps inside a phase are single goals
aimed under roughly 500 LOC, and steps marked **parallel** have no dependency on
their siblings, so different agents can take them independently. Every step
names its verification command. All commands assume
`cd /Users/owebeeone/limbo/gryth-wz/gryth-ui` unless a different directory is
given.

Three stop points require Gianni's decision and are called out where they fall.
Work MUST NOT run past a stop point on assumption.

### Phase 0: Green tree

The foundational milestone. Nothing else in this plan can be verified until the
tree is green, because every later step's verification command is one of the
three gates. Small, and it unblocks everything.

**Step 0.1. Revert the stray `0`.** One character. Not parallel with 0.2 only in
the sense that both should land before the phase is called done.

```sh
git -C /Users/owebeeone/limbo/gryth-wz/gryth-ui checkout -- \
  packages/plugins/chat/src/groups.test.ts
pnpm lint
```

Expected: `pnpm lint` exits 0 with no output.

**Step 0.2. Resolve `@owebeeone/taut-shape`** by the option chosen in section
1.3, recommended (a).

```sh
pnpm install && pnpm test && pnpm lint && pnpm build
```

Expected: all four succeed; `pnpm test` reports 10 of 10 test files passing.

**Step 0.3. Confirm the lock is honest.** Guards against the tree being green
locally and red in CI.

```sh
pnpm install --frozen-lockfile
```

Expected: exits 0 and reports the lockfile is up to date.

**Stop point D1, at the end of Phase 0.** The commit adjudication from section
2: one commit or two, whether to keep or drop the wyred mount, whether
`glp-0006-p1s4-gryth-panels` is the right branch, and whether the false claims
in `GlialWorkspaceNameCutover.md` get amended in the same commit. Phase 1 can be
*investigated* before D1 but MUST NOT be committed across it, because the lock is
shared.

### Phase 1: Settle the topology

Foundational for the actual ask. A new plugin family added on top of the
two-glial split will either pick a side silently or make the split worse, and
either way the resulting bug will be blamed on the new plugin. Fix the ground
first.

**Step 1.1. Make Glial a singleton.** Mirror what already works for grip: add
`"@owebeeone/glial-runtime"` to the `overrides` block in `pnpm-workspace.yaml`
and to `resolve.dedupe` in `vite.config.ts`, then decide which of the two
checkouts wins. Note this is not purely mechanical: (R) the two checkouts are at
different commits (`da06989` versus `0dfe4b9`), so unifying them is a real
choice about which Glial revision gryth runs, and `glade-decl-ts` diverges the
same way (section 6.5). Budget well under 500 LOC of manifest change, but expect
the *decision* to take longer than the edit.

```sh
pnpm install && pnpm why @owebeeone/glial-runtime && pnpm test && pnpm build
```

Expected: `pnpm why` shows exactly **one** resolution of
`@owebeeone/glial-runtime`, where today it shows two.

**Step 1.2. Correct `GlialWorkspaceNameCutover.md`.** **parallel.** Fix the two
false claims identified in the TL;DR and section 2, and either re-verify or
re-word the acceptance-evidence paragraph. Documentation only.

```sh
grep -n "singleton\|clean Glial revision" \
  /Users/owebeeone/limbo/gryth-wz/gryth-ui/dev-docs/GlialWorkspaceNameCutover.md
```

Expected: the surviving text matches what `pnpm why` reports after step 1.1.

**Step 1.3. Deal with the dead `glial-dev` symlinks.** **parallel.** Repoint
`gryth-wz/dev-docs/{DecisionLog.md,StackMap.md,glade,grip-share}` at the live
`glade-wz` documents, or remove them and leave a note. (V) They currently
resolve and silently serve content frozen since 2026-06-17.

```sh
ls -l /Users/owebeeone/limbo/gryth-wz/dev-docs/ | grep '\->'
```

Expected: no remaining link into `glial-dev`.

**Stop point D2, at the end of Phase 1.** The hosting decision from sections 3.3
and 3.5: in-repo package (A) or new external TS member in `gyld-wz` (B), and
which mechanism. Phase 2 cannot start without it.

### Phase 2: Host the plugin family

The milestone that actually answers the question. Assumes option (A) from
section 3.5; if Gianni picks (B) at D2, step 2.1 is replaced by a `gwz repo
add` in `gyld-wz` plus the wyred-pattern wiring listed in section 3.3 option
(i), and step 2.3 becomes mandatory rather than conditional.

**Step 2.1. Scaffold an empty plugin package that registers and opens.** A
`package.json` depending on `@grythjs/plugin-api` only (section 5.3), an
`index.ts` calling `addEntry` with one `ToolDef` whose `windowComponent` renders
a placeholder, and a dependency line in the root manifest. Nothing else. This is
deliberately a hello-world: its purpose is to prove the mount, not to be the
plugin. Well under 500 LOC.

```sh
pnpm install && pnpm test && pnpm lint && pnpm build
```

Expected: all green, and the tool appears in `allTools`. Run the **full** test
suite, not a filtered one, per the eager-glob caution in section 3.4.

**Step 2.2. Add a registration test.** **parallel** with 2.3 and 2.4. Mirror
`packages/plugins/workspace/src/workspace.test.ts`, which (R) already asserts
that a plugin registers its tool under its own grip and that the chrome finds it
via `allTools`. Small.

```sh
pnpm test
```

Expected: the new suite passes; total test-file count increases by one.

**Step 2.3. Close the gate gap.** **parallel.** Only needed if D2 chose (B). Per
section 3.6, either vendor `scripts/no-react-state.test.mjs` into the external
repository's own test run, or teach the script the new root.

```sh
node scripts/no-react-state.test.mjs
```

Expected: `OK: no unapproved React hooks found`, and the external plugin's own
sources are genuinely covered by one of the two runs.

**Step 2.4. Decide placement deliberately.** **parallel.** Per section 5.2,
`ToolDef.role` is ignored, so a new tool falls back to the `stage` area. If the
Gyld tools want a specific foundation area, add entries to the `designate` map
in `packages/desktop/src/foundations.ts`. Tiny, but easy to miss and confusing
when missed.

```sh
grep -n "designate" -A 12 packages/desktop/src/foundations.ts && pnpm test
```

Expected: the new tool ids appear in `designate`, or a conscious decision is
recorded to accept the `stage` fallback.

### Phase 3: Only the shell affordances actually needed

**parallel with Phase 2 once Phase 1 is done**, and deliberately minimal. This
phase exists to be *declined* where possible. Do not start a step here unless
the Gyld design says it needs it; each one is shell surgery on behalf of one
consumer.

**Step 3.1. `Tab.Title`.** Introduce the grip and make `TickerStrip.tsx` consume
the already-declared `ToolDef.tabTitle` (section 5.2). Justified only if the
Gyld browser must distinguish two open graphs by tab. Small, but it touches
shared chrome.

```sh
pnpm test && pnpm lint && pnpm build
```

Expected: green, and two windows of the same tool can show different titles.

**Step 3.2. Move any needed shell grip declaration into `@grythjs/plugin-api`.**
**parallel.** If the Gyld plugins need a grip currently declared in
`grips.desktop.ts`, move the declaration rather than importing
`@grythjs/desktop` from a plugin, which is the violation section 5.3 documents.

```sh
grep -rn "@grythjs/desktop" packages/plugins/*/package.json && pnpm build
```

Expected: no new plugin depends on `@grythjs/desktop`.

**Explicitly out of scope for Phase 3:** the `GraphRenderer` seam and pluggable
layout strategies (NF3 and NF4). Per section 4.5, these are a workspace-plugin
refactor, not a hosting prerequisite, and putting them on this critical path
would serialise two independent efforts for no gain.

### Phase 4: Combined demo runbook

Last because it depends on a green, built UI, and because its scope is the least
settled. Nothing here requires Phase 2 or 3; steps 4.2 onward can proceed in
parallel with them once Phase 0 is green.

**Stop point D3, before Phase 4 starts.** The demo scope question from section
6.3 item 6: does "combined gryth demo and glade demo" mean (i) gryth-ui served
by grazel over a real glade node with the `gwz` and chat suppliers, which is the
small and probably intended job, or (ii) that plus the older `glade/demo`
running alongside, which needs a port move and demonstrates an implementation
next to its own prototype. This plan assumes (i) and recommends it.

**Step 4.1. Verify the Rust side still builds.** Everything this document says
about grazel, `glade-node` and `glade-gwz` is read from manifests and CLI
parsing; (U) none of it was executed. Establish the baseline before designing a
runbook on top of it. Check disk first, build the three crates separately, and
do **not** run `cargo clean` or touch `glade-discover` (section 6.5).

```sh
df -h /System/Volumes/Data
cargo build --manifest-path /Users/owebeeone/limbo/glade-wz/glade/node/Cargo.toml
cargo build --manifest-path /Users/owebeeone/limbo/glade-wz/glade-gwz/Cargo.toml
cargo build --manifest-path /Users/owebeeone/limbo/glade-wz/grazel/Cargo.toml
```

Expected: three successful builds, and free space still comfortably above a few
GB afterwards.

**Step 4.2. Point grazel at a real UI.** (R) `grazel/ui/index.html` is a
placeholder and nothing builds gryth-ui into it. Build the UI and run grazel
with `--ui` aimed at the output, replacing the fallback-by-coincidence of
section 6.1 with real configuration.

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui && pnpm build
curl -s http://127.0.0.1:8080/bootstrap.json
```

Expected: `dist/` is fresh, and the bootstrap endpoint returns
`{"node_ws":"ws://127.0.0.1:9099","mode":...,"name":...}` with the UI served
from the same origin, so the `/bootstrap.json` fetch succeeds instead of falling
back.

**Step 4.3. Write one combined start and stop script.** **parallel** with 4.4.
The actual missing artefact (section 6.3 item 1). It MUST start grazel with an
explicit `--mode`, which has no default, MUST pass an explicit `--data` so two
runs cannot collide, MUST check for a `gwz` binary on `PATH` and say so plainly
if absent rather than silently degrading, and SHOULD keep the existing pidfile
and logfile conventions of `start-demo.sh`. Under 500 LOC comfortably.

```sh
lsof -nP -iTCP:8080 -sTCP:LISTEN && lsof -nP -iTCP:9099 -sTCP:LISTEN
curl -s http://127.0.0.1:8080/bootstrap.json
```

Expected: both ports listening, bootstrap served, and the matching stop script
leaves neither port bound.

**Step 4.4. Record the runbook and retire the stale guide.** **parallel.**
Document the process and port table of section 6.4, and mark
`gryth-wz/dev-docs/GrythDemoProposal.md` superseded, since (R) it cites
`glial-dev` paths that no longer exist and describes a different design.

```sh
grep -rn "glial-dev" /Users/owebeeone/limbo/gryth-wz/dev-docs/
```

Expected: remaining hits are inside explicitly-marked historical sections.

### Dependency summary

```
Phase 0 (green tree)  ──▶ D1 ──▶ Phase 1 (topology) ──▶ D2 ──▶ Phase 2 (host)
                        │                              │
                        │                              └──▶ Phase 3 (affordances, on demand)
                        └──▶ D3 ──▶ Phase 4 (demo, independent of 2 and 3)
```

## 8. What this assessment did not verify

Stated so no reader over-trusts it.

- **Nothing was executed on the Rust side.** No cargo command was run. Every
  claim about grazel, `glade-node` and `glade-gwz` is read from `Cargo.toml`,
  CLI argument parsing, README text and the existence of `target/` directories.
  Step 4.1 exists to close this.
- **No demo was started.** `start-demo.sh` was not run, per the brief. Port
  behaviour and process topology are read from scripts and source.
- **The post-fix state of the tree is unverified.** Proving that `pnpm test`,
  `pnpm lint` and `pnpm build` all pass requires the install in step 0.2, which
  would have rewritten `pnpm-lock.yaml`. The failures were reproduced exactly;
  the fixes were not applied.
- **The `--frozen-lockfile` failure mode is inferred, not observed**, for the
  same reason.
- **The two glial checkouts were compared by commit hash, not by diff.** They
  are known to be different revisions of one repository; the semantic size of
  the divergence was not measured. Step 1.1 should measure it before choosing a
  winner.
- **No Gyld plugin design was read or assessed.** That is another effort's
  topic. The Python-versus-TypeScript data path noted in section 3.5 is flagged
  as an open dependency, not analysed.

## 9. Question map

| Brief question | Section |
|---|---|
| 1. Tree state, hygiene, minimal path to green | 1, and the commit adjudication in 2 |
| 2. Dependency and workspace topology, hosting a new family | 3, especially 3.3 and 3.5 |
| 3. Plugin contract status | 5 |
| 4. Graph rendering readiness | 4 |
| 5. Demo composition | 6 |
| 6. The phased plan and stop points | 7 |
