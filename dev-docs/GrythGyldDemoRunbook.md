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

and step 4 can begin straight away: the desk lands on the glade node by itself,
and **`List` is accepted the first time it is pressed**, because the supplier
gave the bundle root its first build before the script printed that URL. The
script waits for the supplier's `published builds/… (N streams)` line and prints
`first build running (run boot-<session>)` with the elapsed time while it waits.

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

## 2. The bundle root builds itself

There is nothing to do here, and there is nothing to type. **The supplier owns
the first build** (`glade-wz/glade-gyld/README.md`, "The first build is the
supplier's own"): the moment it is serving it reads the bundle root, and

- **on an empty root** — a fresh `--data` — it lays the stage and runs the first
  build itself as run `boot-<session>`, while it goes on serving. Four of its lines say
  so, in this order:

  ```
  [gyld] glade-gyld: first build of .../files/gyld — the bundle root holds none (run boot-<session>)
  [gyld] glade-gyld: serving; SIGTERM/SIGINT to stop
  [gyld] glade-gyld: the checkout declares fork-a, stream-a, stream-b
  [gyld] glade-gyld: published builds/build-1789366082551 (5 streams)
  ```

  It asks the CHECKOUT which streams the staging repository declares, with the
  checkout's own `capture_decision_stream.discover` — the one line
  `manage_decision_streams.py rebuild` runs before it builds — so the first
  build lists the five a `Rebuild` would and not the two a bare emit gives.

- **on a root that already holds a build** — a second start of the same data
  directory — it publishes that build the moment it attaches, and only the
  `published` line follows `serving`.

Until that publication lands, a verb that needs a bundle is refused with the run
rather than with a flat denial:

```
the first build is in progress (run boot-<session>); nothing has landed yet
```

`fork` and `link` are unaffected throughout: they write an overlay module rather
than a bundle and never needed one.

The `published` line is the census reaching the value shares, which is what a
desk reads. `gyld-ui.py start` waits for it — printing `first build running (run
boot-<session>)` with the elapsed time meanwhile — and does not start the dev server or
print the URL until it is there. So a page opened at that URL never lands on a
root with nothing in it. `status` checks the same thing twice over: that
`latest.json` names a build, and that the log carries the supplier's publication
of THAT build.

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
exactly this way, on a fresh `/tmp/gyld-demo-data` — back when the first build
was still made by hand, so `List` was refused on the empty root and accepted
once it was built. Then: `Rebuild` streamed to `end, exit 0`,
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

The launcher's Gyld windows are named plainly — `Graph`, `Streams`, `Details`,
`Next up` — with decide, compare and diff keeping the names they had
(`dev-docs/ui/GyldUiSimplification.md` 2.3, owner ruling U5 of 2026-09-16). The
tool ids behind them are unchanged, so a stored desk still resolves.

Open `+ Streams` from the launcher. The desk is already on the glade node:
an empty desk adds that root itself on the edge into `live`, so the census is
there with nothing pressed and there is no picker to get past
(`gryth-ui/packages/plugins/gyld/README.md`, "What an empty desk lands on").
`Add glade node` is still in the picker for a desk whose root was removed.

1. **List.** `list: accepted, exit 0, run run-1`, with the build's whole
   `streams.json` in the panel. It runs no host. On a root whose first build is
   still in flight it answers `the first build is in progress (run boot-<session>)`
   instead, and `gyld-ui.py start` has already waited that out.
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
4. **Draw it.** Open `+ Graph`, choose stream `demo-keys` and
   perspective `decisions`. The picture draws from a lens pointer fetched over
   the proxy with its sha256 checked. The chrome says how many boxes this
   stream lists as answerable now, those boxes are drawn bright, and every
   `Open` and `Lean` question carries a glyph inside its own box;
   `Next up only` dims the rest and says in the omission strip how many it
   dimmed.
5. **Answer.** Hover the question's own box: the card over it carries the
   question's text, its alternatives with any recorded lean marked, and
   `Answer`, which moves this window onto that question and opens the decide
   window already on it. (`Decide` in the chrome still opens the same window
   on whatever the browser has focused.) The decide window composes on a GLADE
   root: the supplier publishes each stream's `projection.json` and
   `validation.json` on `gyld.file` as pointers beside the lens pointers, so
   the classes an overlay names are read the same way the picture is. The
   static-root detour this step used to require is gone.

   The question is already chosen: the answer form defaults to the record the
   browser is on, which is the one the card just moved it to. Choose an
   alternative, a principal, a stamp and the ruling text. `Export overlay` puts
   the module in the box and `Submit answer` sends that very text. The composed module carries the stream's
   `gyld-stream-record:` block, and for a FORK it declares `follows: base` and
   `imports: glade_decisions`, which is what the module actually imports and
   what its root subclasses.

   `demo-keys` is a fork of `stream-a`, so its own module restates that
   stream's records, and `Submit answer` is refused there, naming the module
   and the six records it already declares. That is the guard under "One
   answer per stream" below doing its job, and the way past it is the flow
   that guard names: make the stream this ruling belongs to. The card says so
   BEFORE the refusal — its button reads `Answer (needs its own stream)` and
   `Link a stream for this ruling` beside it opens `Streams` with `Link`,
   parent `demo-keys` and a name for this ruling already filled in. Press it,
   `Submit`, `Rebuild`; then hover that question again on the new
   stream and press `Answer`. Its module declares nothing but its
   own root class, which the composed module declares again, so `Submit
   answer` is live: the run streams `end, exit 0`, the supplier writes the
   module and rebuilds, and in the new build `demo-keys-key_custody` validates
   `ok` with `key_custody` reading `Decided`.
6. **Diff.** Open `+ Gyld diff`, choose left `base` and right `demo-keys`. An
   emitted diff is the one bundle file still on no share, so add the build as a
   static root (type `/gyld/builds/<build>` in the set picker of a fresh desk)
   to read one that already exists. The pair is not in the bundle, so the window says so,
   prints the command that writes it, and offers `Request this diff`. Pressing
   it runs the supplier's `diff`, which writes into that bundle's own `diffs/`,
   and the window's watch loop picks the file up and renders it in place.

## Asking with a local model

The ask agent talks to Anthropic's Messages API. It will talk to anything else
that speaks it — the owner's `dabeest` box serves a patched Ollama on that API
— and the switch is configuration, never a code path: the Anthropic default is
untouched and stays the default.

grazel composes the supplier's argv itself and passes **none** of its
`--agent-*` flags, so a running desk cannot be configured with a flag. Two
channels reach it, and `gyld-ui.py` serves both.

**The file.** Under the instance's own bundle root, beside the key:

```sh
D=~/.gyld-ui/instances/5173/files/gyld/agent
mkdir -p "$D"
cat > "$D/config.json" <<'JSON'
{ "base_url": "http://127.0.0.1:11434", "model": "qwen3.8-96k" }
JSON
printf 'ollama\n' > "$D/api-key"; chmod 600 "$D/api-key"
```

The key file is mode-checked and refused if any other account can read it. Its
value is ignored by that server and must be there all the same. The `ollama`
compatibility profile is detected from the base URL; `"compat": "ollama"` names
it explicitly when you want it named. The supplier re-reads this file at every
call, so a model changed here takes effect on the next question — a `restart`
is only needed if the instance was not running.

**The environment.** `start` and `restart` carry `ANTHROPIC_BASE_URL`,
`GYLD_AGENT_MODEL`, `ANTHROPIC_AUTH_TOKEN` and `ANTHROPIC_API_KEY` through to
grazel and say which they carried — the endpoint and the model by value, a key
by name alone. The environment beats the file, so a shell that already exports
`ANTHROPIC_BASE_URL` for something else will win: `env -u ANTHROPIC_BASE_URL
python3 gyld-ui.py start` is how you let the file answer.

Before any of it, the tunnel: `dabeest-tunnel up` is idempotent, and
`curl -s http://127.0.0.1:11434/api/version` answers `0.33.0-dabeest`. The
model tags are `qwen3.8-96k` (the daily driver, 96K, best quality) and
`gemma4:26b` / `claude-dabeest-fast` (fast, 256K). One heavyweight at a time on
that GPU. The full story is
`gollama-wz/gollama/dev-docs/DABEEST-CLIENT.md`.

`status` then carries one more line, and `start` ends with it:

```text
ok    agent endpoint — http://127.0.0.1:11434 (agent/config.json) —
      http://127.0.0.1:11434/v1/models answered
```

Anthropic's own endpoint needs a key even to list models, so for that host the
check only asks whether the host resolves; a 401 there is not a failure. An
instance with no base URL anywhere adds no line at all.

The supplier says what it resolved, once, at attach — never the key:

```text
[gyld] glade-gyld: agent base-url http://127.0.0.1:11434 model qwen3.8-96k
       compat ollama (max_tokens 32768, max input 65536)
```

### What it looks like in the window

Right-click a box, `Ask about this`, type a question, `Ask`. The turn draws as
it always does — citations first, then the prose — with one addition: a
**note** line for anything the call had to do differently. Against dabeest
there is exactly one, every turn:

```text
the ollama endpoint has no /v1/messages/count_tokens, so this turn's input
budget is an ESTIMATE of about 1778 tokens (one per 3 characters of request),
not a count
```

That is the guide's own fact — there is no `count_tokens` there — and the
budget is still checked, against an estimate that says it is one. `strict` on
the draft tool and both `cache_control` breakpoints are ACCEPTED by that
server, so neither is dropped and a `propose_draft` call comes back validated:
a local model makes the same offer, with `drafted_by: qwen3.8-96k` on it, and
`Take this draft` works exactly as before. An endpoint that rejected either
would have the supplier retry without it and say so on the same note stream.

Budgets are smaller here and deliberately so. `max_tokens` defaults to 32768
rather than 64000 because these are thinking models and thought tokens come out
of that budget — too small a cap returns empty content with the answer never
emitted — and the input budget defaults to 65536, which is what is left of
qwen's 98,304-token window once the output is reserved.

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
submits: link (or fork) the stream FOR this ruling, rebuild, and answer there.

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

**No projection on a glade root.** Fixed on 2026-09-14. `projection.json` and
`validation.json` were on no share, so the decide window composed nothing on a
glade root, validation read as absent, and every record window said the stream
carried no such record when the truth was that no record was readable at all.
They now travel on `gyld.file` as pointers, like the lenses. An emitted diff is
still on no share.

**A streaming answer names no build.** `stream_output: true` is answered with
`{ok, run_id, done: false}` and the build directory is only on a synchronous
answer, which every building verb avoids because a build is minutes of Python.
The `end` record on the log carries the exit code and not the directory either.
So the supplier panel's offer to read the build it just made never appears for
a verb that builds. It matters far less now that the records travel on
`gyld.file` — a glade root follows each new build on its own — and still costs
a hand-pointed static root for an emitted diff. Putting `output_dir` on the
log's `end` record would fix that and is a `glade-gyld` change.

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
