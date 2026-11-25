# lib-cli-router

`cli-router` — CLI ergonomics layer over `lib-route` (pattern compiler +
segment trie) and `lib-mri` (argv parser), in pure handl. The argv
counterpart of `lib-http-router` (P-008 / T-142). A CLI is a route map
whose path segments are whitespace-separated argv tokens;
`router(routes)` compiles the map once and returns a pure
`(argv) ~> { code, out }` arrow.

## Layout

| path | role |
| --- | --- |
| `src/index.hnl` | the library — exports `router`, `usage`, `entries`, `run`, `flag`, `has_flag` |
| `src/index.test.hnl` | pure-core coverage (`run/3` is effectful, untested) |
| `pkg.hnl` | manifest — deps `mri`, `proc`, `route`, `stdio` (git, `branch:main`) |
| `scripts/hnl-gate.sh`, `ci.sh` | gate + bupha entry |
| `workflows/gate.yaml` | CI — shared `handl-gates` campaign |

## Contract

- `argv` includes argv[0]; it is dropped before `mri.parse`.
- Dispatch is strict: leftover positionals miss — use `"*rest"` or
  `":args"` for trailing operands. Miss → `{ code: 2, out: <usage> }`.
- Handlers receive `{ req, state, params, entry }`; `opts.mri` passes
  through to `mri.parse`.
- `run(app, argv, opts?)` is the effectful shell (prints via `stdio`,
  returns exit code); `opts.exit` stands in for `posix.exit`.

## Verification

```bash
HNL=~/.local/bin/hnl bash scripts/hnl-gate.sh   # or ./ci.sh under bupha
```

<!-- myco:agent-awareness -->

## Agent awareness

- Session start / any work discussion: run `myco status --prompt`
  before proposing next steps — know the board first.
- User describes new work → offer to capture it (`myco todo add`,
  `issue new`, `proposal new`). Don't wait to be asked.
- Picking up a task: `myco resume <ID>` is the briefing.
- Wrapping up: `myco handoff <ID>` anything resumable; keep statuses
  honest (claimed → in-progress → blocked/done).

<!-- /myco:agent-awareness -->
