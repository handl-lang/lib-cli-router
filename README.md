# lib-cli-router

`cli-router` — CLI ergonomics layer over
[`lib-route`](https://github.com/handl-lang/lib-route) (pattern
compiler + segment trie + `{out,state}` pipeline) and
[`lib-mri`](https://github.com/handl-lang/lib-mri) (argv parser), in
pure handl. Proposal P-008 / task T-142 — the argv counterpart of
`lib-http-router`.

A CLI is a route map whose path segments are whitespace-separated argv
tokens. `router(routes)` compiles the map into a `route` trie **once**
and returns a pure `(argv) ~> { code, out }` arrow.

```handl
from "cli-router" import * as crt

app := crt.router({
  deploy: {
    ":env":        (ctx) ~> "deploying " + ctx.params.env,
    ":env/:region": (ctx) ~> "deploying " + ctx.params.env + "@" +
                     ctx.params.region,
    prod:         (ctx) ~> "deploying prod --force"
  },
  status: (ctx) ~> "ok",
  "*rest": (ctx) ~> "unknown: " + ctx.params.rest   -- catch-all
})

app(["mycli", "deploy", "staging"])        -- { code: 0, out: "deploying staging" }
app(["mycli", "deploy", "prod", "--force"]) -- { code: 0, out: "..." }
app(["mycli", "bogus"])                    -- { code: 2, out: <usage text> }
```

## Route maps

- Nested records recurse; keys join with `"/"` into the pattern string
  (the trie consumes one token per segment, so `"/"` is the segment
  separator regardless of transport).
- Keys `"."` and `""` contribute no segment — `{ ".": h }` is the
  no-argument command.
- A leaf is an arrow (handler), a spec record
  `{ handler, path?, pattern?, name?, doc?, ... }` (`path`/`pattern`
  override the derived pattern; `doc.summary`/`doc.description` feed
  `usage`; extra keys ride along on the compiled entry), or a nested
  map. `"handler"` is a reserved key.
- `"*"`/`"*name"` keys are splats and must be leaves; `*name` binds the
  `"/"`-joined remainder of the token list.

## Dispatch

- `argv` **includes argv[0]** (the program name); it is dropped before
  `mri.parse` sees the tail.
- The positional match is **strict**: leftover positionals beyond the
  route's segments miss — add a `"*rest"` catch-all or `":args"`
  segment when a command takes trailing operands.
- Positionals (mri's `"_"` list) are the dispatch tokens; flags land in
  `ctx.state` (the mri record minus `"_"`) and `ctx.req.flags` (the full
  mri record). `opts.mri` passes through to `mri.parse`
  (`string`/`boolean`/`alias`/`default`/`unknown`).
- Handlers receive
  `{ req: { argv, positional, flags }, state: <flags>, params: <hit>,
     entry: <matched entry> }`.
- Results normalize: `{ code: n, out }` passes through; `{ out }` →
  `{ code: 0, out }`; anything else → `{ code: 0, out: r }`.
- Miss → `{ code: 2, out: <usage text> }` (a `"*"` catch-all hits
  instead when present).
- `opts.middleware` — `(ctx, next) ~> { out, state }` onion steps
  around the terminal (via `rt.pipeline`); middleware sees the
  handler's raw return as `out`.

## usage / run

`usage(routes_or_entries, opts?)` renders help — one line per leaf
pattern plus `doc` summary, `opts.about`/`opts.usage` header,
`opts.footer`/`opts.prog` footer. Ordering is the route map's
sorted-key order (deterministic).

`run(app, argv, opts?) ~[1]~>` is the effectful shell: prints `out` to
stdout (code 0) or stderr (otherwise) via the `stdio` contract and
returns the exit code. `opts.exit`, when an arrow, is invoked with the
code after printing.

> **ARGV GAP:** there is no own-process argv source in handl yet (no
> `posix.argv()`, no host args injection), so `run` takes `argv`
> explicitly — the call site will become `run(app, posix.argv())` once
> that primitive lands. `opts.exit` is the stand-in for `posix.exit`.

## Flag helpers

`flag(ctx, name)` / `has_flag(ctx, name)` read `ctx.state` (the parsed
flags; `has_flag` is presence, not truthiness — `--no-verbose` counts).

## Layout

```
src/index.hnl        the library (exports: router, usage, entries,
                     run, flag, has_flag)
src/index.test.hnl   pure-core coverage (run/3 is effectful, untested)
```

Gate: `HNL=~/.local/bin/hnl bash scripts/hnl-gate.sh`
(or `bash ci.sh` under bupha).
