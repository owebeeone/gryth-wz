# The Gyld write path, end to end

How to stand the whole composition up and drive the Gyld write path from the
gryth desktop: grazel with the `glade-gyld` supplier behind it, `pnpm dev` in
front of it, and `list`, `fork`, `answer` and `diff` submitted from the
windows, with each run's output arriving on the `gyld.output` log share.

Written from a run that actually did it on 2026-09-14. Everything below is what
was typed and what came back, including the two things that do not work the way
you would first expect and the one that lost data before it was fixed.

## Start it with the script

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui
python3 gyld-ui.py start
```

That is sections 1 to 3 below, done and checked, ending on the line

```
URL: http://localhost:5173/
```

and step 4 can begin at `Add glade node` — **`List` is accepted the first time
it is pressed**, because the script lays the bundle root and gives it its first
build itself. The "press `List` first" ordering under section 2 is a property of
the hand-driven path only.

```sh
python3 gyld-ui.py status                # ok/FAIL per check, then working / not working
python3 gyld-ui.py start --port 5180     # a second composition, its own ports and data
python3 gyld-ui.py restart
python3 gyld-ui.py stop [--purge]        # no --port: every instance
```

`start` is idempotent; `--mode built` is the no-proxy variant of section 3;
`--http` and `--node-port` derive from `--port` (5173 → 8080/9099, the ports
printed throughout this document) so two compositions never collide. An
instance's data lives in `~/.gyld-ui/instances/<port>/` and is KEPT across a
stop, because a ruling submitted from a window is written into the bundle root
there; `stop --purge` is what deletes it. The option surface is in
`gryth-ui/README.md`, "Running the Gyld composition".

**The rest of this document is what the script does, and why.** Read it to
drive a piece of the composition by hand, to understand what a check is
checking, or to find out what a refusal means.

## What talks to what

| piece | where | what it is |
|---|---|---|
| `glade-node` | `ws://127.0.0.1:9099` | the node; grazel spawns it and attaches over the wire |
| `grazel` | `http://127.0.0.1:8080` | the composition root: node, suppliers, static paths |
| `glade-gyld` | behind `(ws-razel, gyld.ops)` | the Gyld verbs, run as subprocesses of the Gyld hosts |
| `glade-gwz` | behind `(ws-razel, gwz.ops)` | the sibling supplier, spawned by the same grazel |
| `pnpm dev:gyld` | `http://localhost:5173` | the Gyld-only desktop; proxies `/gyld/` and `/bootstrap.json` to grazel |
| `gyld-ui.py` | `gryth-ui/gyld-ui.py` | starts, checks and stops all of the above as one instance |
| the bundle root | `<data>/files/gyld` | app owned, the supplier's outright; grazel serves it at `/gyld/` |
| the Gyld checkout | `gyld-wz/gyld` | READ ONLY: the hosts are run out of it and nothing is written back |

## Before you start

Three debug binaries have to exist. None of them is built by this runbook:

```sh
ls -l /Users/owebeeone/limbo/glade-wz/glade/node/target/debug/glade-node
ls -l /Users/owebeeone/limbo/glade-wz/grazel/target/debug/grazel
ls -l /Users/owebeeone/limbo/glade-wz/glade-gyld/target/debug/glade-gyld
```

If one is missing, build it with `cargo build` in that member, after checking
there is room:

```sh
df -h /System/Volumes/Data      # stop if less than 5 GiB is free
```

The Gyld hosts need Python 3.13; the system `python3` is 3.10 and they fail on
it. The supplier defaults to `/opt/homebrew/bin/python3.13` and takes
`--python` if yours is elsewhere.

`gyld-ui.py start` checks every one of these before it starts anything — the
three binaries, `pnpm`, `node_modules`, the 3.13, the Gyld checkout and the
5 GiB floor — and prints the fix beside whichever one is not there. It builds
none of them: a missing binary is still `cargo build` in that member, by hand.

## 1. Start grazel, with the gyld leg on

The gyld leg is default off. `--gyld-supplier-bin` is the switch, and it is
also what makes grazel load `apps/gyld-app.glade` beside `apps/grazel-app.glade`.

```sh
cd /Users/owebeeone/limbo/glade-wz/grazel
mkdir -p /tmp/gyld-demo-data
./target/debug/grazel --mode both --data /tmp/gyld-demo-data \
  --http 8080 --node-port 9099 \
  --gyld-supplier-bin ../glade-gyld/target/debug/glade-gyld \
  --gyld-root /Users/owebeeone/limbo/gyld-wz/gyld
```

Six lines out of its log say it worked, in this order:

```
[node] app grazel registered (+11 record(s), 0 unchanged)
[node] app gyld registered (+8 record(s), 1 unchanged)
[node] workspace ws-razel serving
[node] listening 9099
[gyld] glade-gyld: attaching to ws://127.0.0.1:9099 as ws-razel/gyld.ops (...)
[gyld] glade-gyld: serving; SIGTERM/SIGINT to stop
```

`1 unchanged` on the gyld app is the `workspace ws-razel razel` entry both app
files declare; registration is idempotent by diff, so it registers once.
`[gwz] glade-gwz: serving` appears too: the gwz leg is on by default and is not
in the way.

`--data` is app storage. Put it somewhere disposable: the bundle root under it
is written to by every build and nothing is ever built over an existing build,
so it grows.

## 2. Give the bundle root its first build

This step is NOT optional today, and it is the first surprise. The supplier's
bundle root starts empty, and `list`, `rebuild`, `answer`, `ask` and `diff` all
resolve the latest build first, so on a fresh root every one of them answers

```
list: refused, attributed to <principal>
no bundle has been built yet
```

which is correct and is not a fault. `fork` and `link` are the two that need no
build, but they write an overlay module rather than a bundle, so they do not
break the deadlock either: the stream manager's parent picker is filled from the
census, and the census comes from a build.

So make the first build with the Gyld host directly, into the layout the
supplier documents, and point `latest.json` at it:

```sh
BR=/tmp/gyld-demo-data/files/gyld           # the bundle root
cd /Users/owebeeone/limbo/gyld-wz/gyld
PYTHONPATH=src:. /opt/homebrew/bin/python3.13 -B scripts/emit_decision_streams.py \
  --repository $BR/stage --output $BR/builds/build-seed --architecture
printf '{"output_dir":"builds/build-seed"}' > $BR/latest.json
```

`$BR` does not exist until the supplier has been asked for something once, so on
THIS path press `List` in the UI first (step 4) or run the demo in this order:
start grazel, open the desk, add the glade root, press `List`, watch it refuse,
then seed. Run it before the root is laid and you get the failure that named it:

```
No such file or directory: .../stage/examples/glade-decisions.gyld.py
```

`$BR/stage/examples` is a symlink to `$BR/overlays`, which the supplier seeds
from the read-only checkout's `examples/`, so the seed build captures the
committed streams and nothing else.

`gyld-ui.py start` does not have this ordering problem: it lays the bundle root
itself, with a mirror of the supplier's own `ensure_stage`
(`glade-gyld/src/bundle.rs`), and then seeds. The mirror is create-if-absent and
idempotent, laying exactly the tree the supplier lays, so the supplier's own lazy
pass afterwards finds nothing to do and a written overlay still wins.

The script's seed also asks the checkout which streams it declares and passes
each one to the host as `--stream`, the way `manage_decision_streams.py rebuild`
does. The bare command above emits `base` and `architecture` only, two streams;
with the declared streams it emits five, which is what the census reads after
step 4's `Rebuild` either way.

A supplier that bootstrapped its own first build, or a `rebuild` that accepted
an empty root, would remove this step for the hand-driven path too. That is a
change in `glade-gyld` and is not made here.

## 3. Start the desktop

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui
pnpm dev:gyld
```

`http://localhost:5173`. `dev:gyld` is the GYLD-ONLY target (`entries/gyld`,
`vite.gyld.config.ts`): the same desktop whose whole plugin list is
`@grythjs/plugin-gyld`, so the launcher offers the seven Gyld windows and
`+ Welcome` and nothing unrelated. `pnpm dev` is the full desktop and drives
this runbook identically; the two differ only in their plugin list.

The dev server proxies TWO of grazel's paths onto its own origin, both at
`GRAZEL_URL` (default `http://127.0.0.1:8080`).

`/gyld/` is what makes a lens pointer resolve: a `gyld.lens` value is a
`{path, digest, bytes}` pointer whose `path` is grazel's, and without the proxy
a glade root lists streams and draws nothing. That proxy key is a regex, so
`/gyld-bundle/` and `/gyld-evaluator/`, which are the dev server's own mounts
over the Gyld artifacts directory, are untouched.

`/bootstrap.json` is grazel's session placement, and it is what tells the page
which node to attach to. It was NOT proxied until 2026-09-14, so the page fell
back to `ws://127.0.0.1:9099` — which happened to be right for one composition
on the default ports, and wrong for any second one: its desk would have attached
to the FIRST composition's node. Proxied, `gyld-ui.py start --port 5180` gives a
page that opens `ws://127.0.0.1:9106`, its own node, which is how that was
checked. With no grazel behind the proxy the fetch does not answer `ok` and the
`ws://127.0.0.1:9099` fallback applies exactly as before, with the proxy error
in the browser console.

### Or: let grazel serve the desktop, and have no proxy at all

Build the target and hand the directory to grazel in step 1, and the page and
the bundle root are one origin:

```sh
cd /Users/owebeeone/limbo/gryth-wz/gryth-ui
pnpm build:gyld                                    # -> dist-gyld/
```

then add to the grazel command of step 1

```
  --ui /Users/owebeeone/limbo/gryth-wz/gryth-ui/dist-gyld
```

and open `http://127.0.0.1:8080/` instead. No `pnpm dev` runs at all; the lens
is fetched from `/gyld/...` on the page's own origin. Driven on 2026-09-14
exactly this way: fresh `/tmp/gyld-demo-data`, `List` refused on the empty
root, seeded per step 2, `List` accepted, `Rebuild` streamed to `end, exit 0`,
`glade node: ready - watching` with 5 streams in the stream manager, and stream
`base` perspective `decisions` drawn in the browser from
`/gyld/builds/<build>/streams/base/lenses/decisions.lens.json`. The full
desktop does the same with `pnpm build` and `--ui dist`.

Remember to rebuild the directory after a UI change: grazel serves the bytes in
it and knows nothing about the source.

`gyld-ui.py start --mode built --port 8090` is this variant: `--port` is then
grazel's own HTTP port, `dist-gyld` is built if it is not there (`--build`
forces it), and no vite runs at all.

## 4. Drive it

Open `+ Gyld streams` from the launcher and press `Add glade node`.

1. **List.** Answers `list: refused ... no bundle has been built yet` on a
   fresh root, and after step 2 answers `list: accepted, exit 0, run run-1`
   with the build's whole `streams.json` in the panel. It runs no host.
2. **Rebuild.** A streaming run: the answer is `rebuild: accepted, run run-2`
   and the run's lines arrive on `gyld.output` keyed by that run, ending
   `end, exit 0`. When it lands, the census arrives on the shares and the
   window redraws itself into the stream tree: `5 streams`, `glade node: ready
   - watching`. Nothing is reloaded.
3. **Fork.** In the new-stream form choose `Fork`, parent `stream-a`, name
   `demo-keys`. `Export command` writes
   `PYTHONPATH=src:. python3 -B scripts/manage_decision_streams.py fork stream-a demo-keys`
   and `Submit` sends the same two operands as the `fork` verb. The run streams
   the host's own record of what it wrote and closes `end, exit 0`. `Rebuild`
   again and the tree says `6 streams`.
4. **Draw it.** Open `+ Gyld browser`, choose stream `demo-keys` and
   perspective `decisions`. The picture draws from a lens pointer fetched over
   the proxy with its sha256 checked.
5. **Answer.** Press `Decide`. On a GLADE root both forms refuse with
   `this stream carries no projection here`, because the supplier publishes
   `stream.json`, `decide-now.json` and the lens pointers and NOT
   `projection.json`, and the classes an overlay names are declared in the
   projection. That refusal is the second surprise and it is honest.

   To answer, read the build as a static root instead. The build directory is
   under grazel's static path, so in the set picker of a fresh desk type

   ```
   /gyld/builds/build-1789341052425
   ```

   (whatever `latest.json` names), press `Add static root`, and the same
   windows read the whole build, projection and validation included. `Gyld.Ops`
   does not depend on the root: it is the glade session, so a static root
   submits exactly as a glade root does.

   FRESH desk is literal: the picker is what a Gyld window draws while the desk
   holds no root at all, and there is no other place to add or drop one, so a
   desk already on the glade node shows no picker and neither does a second
   desktop or another workspace, which share the same root set. Reload the page
   to get the picker back. Every later build is added the same way, by reload.

   Then choose the question, an alternative, a principal, a stamp and the
   ruling text. `Export overlay` puts the module in the box and `Submit answer`
   sends that very text. The composed module carries the stream's
   `gyld-stream-record:` block, and for a FORK it declares `follows: base` and
   `imports: glade_decisions`, which is what the module actually imports and
   what its root subclasses.

   `demo-keys` is a fork of `stream-a`, so its own module restates that
   stream's records, and `Submit answer` is refused there, naming the module
   and the six records it already declares. That is the guard under "One
   answer per stream" below doing its job, and the way past it is the flow
   that guard names: make the stream this ruling belongs to. In
   `+ Gyld streams` choose `Link`, parent `demo-keys`, name
   `demo-keys-ruling`, `Submit`, `Rebuild`; add THAT build as a static root
   and press `Decide` on the new stream. Its module declares nothing but its
   own root class, which the composed module declares again, so `Submit
   answer` is live: the run streams `end, exit 0`, the supplier writes the
   module and rebuilds, and in the new build `demo-keys-ruling` validates
   `ok` with `key_custody` reading `Decided`.
6. **Diff.** Open `+ Gyld diff`, add the same static root, choose left `base`
   and right `demo-keys`. The pair is not in the bundle, so the window says so,
   prints the command that writes it, and offers `Request this diff`. Pressing
   it runs the supplier's `diff`, which writes into that bundle's own `diffs/`,
   and the window's watch loop picks the file up and renders it in place.

## What a run looked like

```
run-1  list      accepted, exit 0        streams.json in the panel
run-2  rebuild   accepted (streaming)    end, exit 0; census 5 streams
run-3  fork      accepted (streaming)    wrote glade-decisions-demo-keys.gyld.py
run-4  rebuild   accepted (streaming)    end, exit 0; census 6 streams
run-5  answer    accepted (streaming)    wrote the overlay, rebuilt, validation ok
run-6  diff      accepted (streaming)    wrote diffs/base..demo-keys.json
```

Console errors during all of it: `404 (Not Found)`, repeating every four
seconds, for the diff path while the diff did not exist. That is the store's
watch loop re-requesting a path that answered 404, which the plugin README
already records as a known limit. They turned into 200s the moment the file
was written.

## The three things that bit

**One answer per stream.** The supplier's `answer` and `ask` write the overlay
MODULE, whole, and the decide window composes a module holding the ONE record
the draft adds. Submitted rather than merged, that text deletes every other
record the module declared. The first run of this demo did exactly that: the
fork `demo-keys` went from 25 questions and 3 rulings to 24 and 1. The window
now refuses a submit where the stream's own module already declares records,
naming the module and the count, and Export is untouched because merging by
hand is what it is for. So the flow is one fork or link per ruling, which is
what specification section 6.1 describes anyway, and it is the path that
submits: link (or fork) the stream FOR this ruling, rebuild, read the new build
as a static root, and answer there.

The records counted are the ones the module places under its root,
`<module>:<Root>.<member>`. The ROOT CLASS does not count: its slot is
`<module>:<Root>` with no member, every generated overlay declares it, and the
composed module declares it again under the name the stream record registers,
so nothing is lost by rewriting it. The first cut of the guard counted it, and
because a fresh link declares exactly that one class it refused every fork and
every link with "already declares 1 record" - the advice and the guard closed
on each other and no stream could be answered at all. A re-run on 2026-09-14
found that; the guard now counts members only, and refuses separately, naming
both, when the root class the module declares is not the root the stream
registered.

**No projection on a glade root.** Two files of a bundle are on no share,
`projection.json` and `validation.json`, and neither is an emitted diff. The
decide window needs the projection, so it composes nothing on a glade root and
says so. Reading the build as a static root is the answer, and that is what
step 5 does.

**A streaming answer names no build.** `stream_output: true` is answered with
`{ok, run_id, done: false}` and the build directory is only on a synchronous
answer, which every building verb avoids because a build is minutes of Python.
The `end` record on the log carries the exit code and not the directory either.
So the supplier panel's offer to read the build it just made never appears for
a verb that builds, and a static root has to be pointed at the new build by
hand. Putting `output_dir` on the log's `end` record, or publishing the build
on a value share of its own, would fix it and is a `glade-gyld` change.

## Stopping

```sh
python3 gyld-ui.py stop            # this instance's two groups, then a check that they are gone
python3 gyld-ui.py stop --purge    # and delete its data directory
```

By hand:

```sh
pkill -f 'target/debug/grazel'     # tears the node and both suppliers down with it
pkill -f 'vite'                    # or Ctrl-C in the pnpm dev / dev:gyld terminal
rm -rf /tmp/gyld-demo-data         # the builds are large and disposable
```

A SIGINT or SIGTERM to grazel tears every child down; the node exiting takes
grazel with it, nonzero, with the tail of the node's stderr.

**The stale lock.** The node's `instance.lock`
(`<data>/sys/sys/grazel/instance.lock`, glade/node `sysdir.rs`) is created
O_EXCL and removed on drop, so a node that was killed rather than asked to stop
leaves one behind and the next start refuses with

```
instance already locked: .../instance.lock
```

The file holds the writer's pid, which is what tells a stale lock from a live
one. `gyld-ui.py` reads it: a dead pid is removed and the removal is printed, a
live one is reported as the running node it is, and `stop` clears the lock its
own SIGTERM left. By hand, check the pid with `ps -p` and delete the file only
when it is gone.

## Where the pieces are written down

- `gryth-wz/gryth-ui/gyld-ui.py` and `gryth-ui/README.md`, "Running the Gyld
  composition", for the script that does all of the above.
- `glade-wz/grazel/README.md`, "Composed suppliers" and "The gyld static path".
- `glade-wz/glade-gyld/README.md`, "The bundle root", "The allow-list",
  "Long-op output" and "Results".
- `gryth-wz/gryth-ui/packages/plugins/gyld/README.md`, "The glade node as a
  root", "Submitting to the supplier" and "Running the write path".
- `gyld-wz/dev-docs/ui/GyldGrythPlugins.md` sections 4.7 and 6, and step 4.4.
- `gyld-wz/gyld/examples/README.md`, "Which streams exist, and how a host
  knows", for the `gyld-stream-record:` block every overlay carries.
