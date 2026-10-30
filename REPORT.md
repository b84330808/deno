# Deno bug fix report: descriptive error for non-string `env` values in `Deno.Command`

Prepared in a cloud session on 2026-10-30. Nothing was opened, edited or
commented on upstream (`denoland/deno`). The upstream issue number appears only
in this file and in the suggested PR body below, never in a commit message, so
pushing these branches doesn't add a cross-reference to the upstream issue.

## At a glance

| Item         | Value                                                                                                                                                                       |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Issue        | https://github.com/denoland/deno/issues/18556: "Deno.Command().spawn() fails with unclear error message on undefined envs"                                                  |
| Issue status | Open. Labels `bug` and `public API`, issue type Bug. Labeled by bartlomieju (Member). Member crowlKats commented on the expected fix (quoted below).                         |
| Fix branch   | `b84330808/deno` → `claude/eager-turing-s0as7p`                                                                                                                             |
| Fix SHA      | `3d891976393d44f3ce99816d28686d4135c3c720` (one commit)                                                                                                                     |
| Base         | `upstream/main` = `e7b39ebd55c85aaf6a29290c1f5a99e2e21e09dd` ("revert: type-check with native tsc by default (#37057)", 2026-10-26)                                          |
| Commit author | `b84330808 <b84330808@gmail.com>` with trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`                                                                   |
| Compare / PR | https://github.com/denoland/deno/compare/main...b84330808:deno:claude/eager-turing-s0as7p                                                                                   |
| PR title     | `fix(ext/process): clear error for non-string env values in Deno.Command`                                                                                                   |
| Files        | `ext/process/40_process.js` (+18/-3), `tests/unit/command_test.ts` (+19)                                                                                                    |

## One thing to decide before opening the PR: a stale assignment

On 2023-04-02 bartlomieju assigned #18556 to **levex** (Levente Kurusa, a Deno
team member at the time). There has been no activity from them on the issue
since then:

- levex's last PR to `denoland/deno` is #19279, merged 2023-06-05. All 16 of
  their PRs are closed, and none touches `Deno.Command` or env handling.
- The issue's Development section reads "No branches or pull requests". The
  timeline has no cross-references, linked PRs or fork commits.
- The last comment is from 2023-05-31, and the last activity of any kind is a
  label removal on 2024-06-20.

I treated this as a stale assignment, not an active claim, because no PR, branch
or commit exists in any state. If you'd rather avoid even a stale assignment, do
not open the PR. Every other unclaimed `bug` issue I checked was weaker; see the
screening log at the end.

## The bug

Repro from the issue, still reproducible on `main` (`e7b39ebd`):

```ts
new Deno.Command("deno", { args: ["run", "x.ts"], env: { HELLO: undefined } })
  .spawn();
```

```
error: Uncaught (in promise) TypeError: serde_v8 error: invalid type; expected: string, got: undefined
    at spawnChildInner (ext:deno_process/40_process.js:206:17)
    at spawnChild (ext:deno_process/40_process.js:232:10)
    at Command.spawn (ext:deno_process/40_process.js:828:12)
```

The error never mentions `env` or the variable name. Numbers and `null` fail the
same way ("got: Number"), and so do `output()` and `outputSync()`.

Maintainer guidance on the issue, from crowlKats (Member) on 2023-05-31:

> @waterbang even when fixed, this will still error. I wouldnt recommend going
> back to Deno.run as its deprecated. if you need to do this, i'd recommend to
> filter out entries that have undefined values.

So the fix keeps the error and makes it descriptive. `CommandOptions.env` is
typed `Record<string, string>`.

## The fix

The fix adds a small `envEntries(env)` helper in `ext/process/40_process.js`,
used by `spawnChildInner` (`spawn()`), `spawnInner` (`output()`) and
`spawnSyncInner` (`outputSync()`). The `Deno.spawn` / `spawnAndWait` /
`spawnAndWaitSync` helpers go through `Deno.Command`, so they're covered too.
After the fix:

```
TypeError: Cannot set environment variable 'HELLO': value must be a string, received undefined
```

`null` is reported as `null` (not `object`), and other values use `typeof`.

The fix leaves these paths alone on purpose:

- **`nodeSpawnChild` / `nodeSpawnSyncChild` (`node:child_process`).** These
  already normalize env like Node does (`normalizeSpawnArguments` skips
  `undefined` and builds `KEY=value` strings), so they never pass non-strings.
- **`run` (`Deno.run`, soft-removed in Deno 2).** Its types are gone, and the
  issue is about `Deno.Command`.

```diff
+function envEntries(env) {
+  const entries = ObjectEntries(env);
+  for (let i = 0; i < entries.length; ++i) {
+    const { 0: key, 1: value } = entries[i];
+    if (typeof value !== "string") {
+      throw new TypeError(
+        `Cannot set environment variable '${key}': value must be a string, received ${
+          value === null ? "null" : typeof value
+        }`,
+      );
+    }
+  }
+  return entries;
+}
...
-    env: ObjectEntries(env),
+    env: envEntries(env),          (x3: spawnChildInner, spawnInner, spawnSyncInner)
```

Regression test `commandEnvValueNotString` in `tests/unit/command_test.ts`
checks `undefined`, `null` and `1` against `spawn()`, `output()` and
`outputSync()`. Each call must throw a `TypeError` with the message above.
`output()` throws synchronously, as it does today for `NotFound`, so the test
uses `assertThrows`, like the existing `commandNotFound` test.

## Tests: fail before, pass after

Setup: a debug build (`cargo build --bin deno`, 10.5 min on 4 cores) of
`upstream/main` `e7b39ebd`, plus `git submodule update --init tests/util/std`.
I ran the tests with the repo's harness:
`cargo test -p unit_tests --test unit -- command_test`, which is what
`./x test-unit command_test` runs.

**Environment caveat (local only, not committed).** This container's kernel has
no IPv6, so `test_server` could not bind its `[::1]` npm-registry listeners, and
the harness waited forever for 39 "ready" lines. For these runs only, I patched
`tests/util/server/servers/hyper_utils.rs` so an IPv6 bind failure still prints
`ready:`. I reverted it before committing: the fix commit touches only the two
files above. `command_test` doesn't use those servers.

### Before: fix reverted, test present

I ran `git stash push ext/process/40_process.js` and used the binary built from
pristine `e7b39ebd`. The new test fails, and the other 70 tests in the file
pass:

```
test unit::command_test ... fail (12943ms)
...
 ERRORS

commandEnvValueNotString => ./tests/unit/command_test.ts:811:6
error: AssertionError: Expected error message to include "Cannot set environment variable 'HELLO': value must be a string, received undefined", but got "serde_v8 error: invalid type; expected: string, got: undefined".
    throw new AssertionError(msg);
          ^
    at assertIsError (file:///home/user/deno/tests/util/std/assert/is_error.ts:63:11)
    at assertThrows (file:///home/user/deno/tests/util/std/assert/throws.ts:96:7)
    at commandEnvValueNotString (file:///home/user/deno/tests/unit/command_test.ts:823:7)

 FAILURES

commandEnvValueNotString => ./tests/unit/command_test.ts:811:6

FAILED | 70 passed | 1 failed | 4 ignored (9s)

error: Test failed
...
1 failed of 1
error: test failed, to rerun pass `-p unit_tests --test unit`
```

### After: fix applied, binary rebuilt

```
$ cargo test -p unit_tests --test unit -- command_test
     Running unit

test unit::command_test ... ok (13179ms)

1 tests passed
```

For per-test detail, I ran the same `deno test` arguments the harness passes
(`tests/unit/mod.rs`), filtered to the env tests:

```
$ ./target/debug/deno test --config tests/config/deno.json --no-lock --unstable-net \
    --unstable-vsock --location=http://127.0.0.1:4545/ --no-prompt -A \
    tests/unit/command_test.ts --filter "/commandEnv|commandSyncEnv/"
Check tests/unit/command_test.ts
running 3 tests from ./tests/unit/command_test.ts
commandEnv ... ok (64ms)
commandSyncEnv ... ok (48ms)
commandEnvValueNotString ... ok (2ms)

ok | 3 passed | 0 failed | 72 filtered out (123ms)
```

### Related suites, with the fix applied

| Suite                                                                          | Result                                                                                                                                                                       |
| ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cargo test -p unit_tests --test unit -- process_test`                         | ok                                                                                                                                                                           |
| `cargo test -p unit_tests --test unit -- tty_color_test`                       | ok                                                                                                                                                                           |
| `cargo test -p unit_node_tests --test unit_node -- child_process_test`         | ok                                                                                                                                                                           |
| `cargo test -p unit_tests --test unit -- os_test`                              | 29 passed, 2 failed: `getUid` and `getGid` assert `uid > 0` / `gid > 0`, and this container runs as root (uid 0). Unrelated to this change, and they don't use `Deno.Command`. |

## Format and lint

- **`./tools/format.js` could not run here.** It runs `npm:dprint@0.47.2`, which
  downloads its plugins from `plugins.dprint.dev`, and this sandbox's network
  policy blocks that host ("Proxy failed to connect"). As an equivalent check, I
  ran the same dprint binary (0.47.2) with the exact TypeScript plugin version
  pinned in `.dprint.json` (`typescript-0.96.0`, taken from the npm package
  `@dprint/typescript@0.96.0`) and the repo's TypeScript settings
  (`"deno": true, "functionExpression.flatIife": true`).
  `dprint check ext/process/40_process.js tests/unit/command_test.ts` exited 0.
  As a negative control, a copy with a deliberate extra space was flagged, so
  the check does catch misformatting. **Please run `./tools/format.js` (or
  `./x fmt`) locally before opening the PR to confirm.**
- **`./tools/lint.js --js` exited 0.** It includes `deno lint` with the
  `prefer_primordials` plugin over `ext/**/*.js`. I also linted the two changed
  files directly with the same configs: "Checked 1 file", no findings.
- **No Rust code changed,** so the full `tools/lint.js` (clippy) isn't needed,
  per `CLAUDE.md`.

## Policy check

Sources: `.github/CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`,
`CLAUDE.md`, `.github/copilot-instructions.md` and `tools/verify_pr_title.js`,
all at `e7b39ebd`. I couldn't fetch https://docs.deno.com/runtime/contributing/,
which the template links as `https://docs.deno.com/runtime/manual/references/contributing`,
because this sandbox can't resolve `docs.deno.com`. Please take a quick look at
it locally.

- **AI use is allowed with disclosure.** `CONTRIBUTING.md` says, verbatim:

  > **AI-assisted contributions:** If you use AI tools (e.g. Copilot, ChatGPT,
  > Claude, Cursor, etc.) to help write your contribution, **you must disclose
  > this in your PR description**. There is no penalty for using AI tools, but
  > PRs will be rejected if there is suspicion of undisclosed AI usage.

  The PR template says, verbatim:

  > IMPORTANT: If you used AI tools (e.g. Copilot, ChatGPT, Claude, Cursor,
  > etc.) to help write this PR, you MUST disclose it in the PR description. PRs
  > will be rejected if there is suspicion of undisclosed AI usage.

  Item 6 of the template adds: "If you used AI tools to help write this PR, you
  MUST disclose it below. PRs will be rejected if there is suspicion of
  undisclosed AI usage."

- **Exact wording required:** none. The policy only requires the disclosure to
  be in the PR description. The body below has an explicit "AI disclosure"
  section and ends with "I used Claude Code to write this change."
- **No ban on agent-made PRs, and no personal-testing attestation.** Nothing in
  these files forbids agent-made PRs or makes the author attest to testing it
  personally, so there was no reason to stop.
- **Spam rule:** "If you create multiple issues or PRs or create multiple
  comments on issue or PRs that look AI generated, that are not substantial,
  your account may be banned." Keep this to one focused PR.
- **Other template requirements:**
  - Descriptive title with a valid prefix: `fix(...)` passes
    `tools/verify_pr_title.js`.
  - A referenced issue: `Fixes #18556`.
  - Tests: included.
  - `./x fmt` and `./x lint`: see above.
- **`CLAUDE.md` workflow:** never force-push to the PR branch, and keep changes
  minimal.

## PR title

```
fix(ext/process): clear error for non-string env values in Deno.Command
```

## PR body (copy-paste)

The template is only an HTML comment with instructions and has no section
headings to fill in. This body covers each item: a referenced issue, tests, and
an AI disclosure.

````markdown
Fixes #18556

Passing a non-string value in the `env` option of `Deno.Command` (for example
`env: { HELLO: undefined }`) fails with an opaque deserialization error that
doesn't mention `env` or which variable is wrong:

```
error: Uncaught (in promise) TypeError: serde_v8 error: invalid type; expected: string, got: undefined
    at spawnChildInner (ext:deno_process/40_process.js:206:17)
    at spawnChild (ext:deno_process/40_process.js:232:10)
    at Command.spawn (ext:deno_process/40_process.js:828:12)
```

As noted in the issue, this should still be an error (`env` is typed as
`Record<string, string>`), but a descriptive one. This PR validates the `env`
values in `Deno.Command#spawn()`, `#output()` and `#outputSync()` before calling
into the op, and throws a `TypeError` that names the variable and the received
type:

```
TypeError: Cannot set environment variable 'HELLO': value must be a string, received undefined
```

The `node:child_process` code paths are unchanged. They already normalize env
values (skipping `undefined`) before spawning.

### Tests

Added `commandEnvValueNotString` to `tests/unit/command_test.ts`. It checks
`undefined`, `null` and a number against `spawn()`, `output()` and
`outputSync()`. It fails on `main` with the `serde_v8` message and passes with
this change. The rest of `command_test.ts` and `process_test.ts`, plus
`unit_node/child_process_test.ts`, still pass.

### AI disclosure

This PR was prepared with an AI coding assistant (Claude Code). It wrote the
fix, the regression test and this description, and ran the tests listed above.
I used Claude Code to write this change.
````

## Duplicate search (all PR states)

I ran these searches on github.com against `denoland/deno` on 2026-10-30, and
re-checked the first two right before writing this report.

| Query                                                              | Result                                                                                                                                                                                      |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Issue page #18556 (header, sidebar Development, timeline)          | No linked PRs or branches ("No branches or pull requests"), no cross-references, no fork commits. Assignee: levex (2023). Last activity 2024-06-20.                                        |
| `is:pr 18556`                                                      | **0 open, 0 closed**                                                                                                                                                                        |
| `is:pr Command env undefined`                                      | 17 (1 open, 16 closed): none related. Open one: #28002 (ext/canvas lcms2). The rest are unrelated feature or fix PRs, e.g. #35317, #31380, #31187.                                           |
| `is:pr Deno.Command env`                                           | 11: none related. Open: #36442 (node_fs fd metadata), #36278 (cwd via runtime FileSystem). Closed or merged: #36433, #36224, #34061, #33587, #31380, #28291, #21939, #17140, #1156.          |
| `is:pr ext/process env`                                            | 88 total. I reviewed the first page (25), and none validates `Deno.Command` env values. The closest is #36999, "fix(ext/node): clear parent env in child_process sync APIs" (open). Judging by its title it concerns `node:child_process`, not validating `Deno.Command` env values. |
| `is:pr undefined env spawn`                                        | 10, all closed or merged, none related (#35317, #34416, #33815, #33555, #32805, #32755, #31380, #3863, #2294, #1134).                                                                       |
| `is:pr environment variable must be a string`                      | 12, all closed, none related.                                                                                                                                                               |
| `is:pr serde_v8 ExpectedString`                                    | 2, both merged long ago and unrelated: #14027 (2022, adds argument position to serde_v8 errors) and #10422 (2021).                                                                          |
| `is:pr "unclear error"`                                            | 0                                                                                                                                                                                           |
| `is:pr author:levex` (the assignee)                                | 16, all closed, the last merged 2023-06-05. None about env or `Deno.Command`.                                                                                                               |
| Issues and PRs: `Deno.Command env undefined`                       | Only #18556 itself, plus unrelated #31380, #19759, #17070, #15427, #11016.                                                                                                                  |

## Candidate screening log (why other issues were skipped)

126 issues are open with the `bug` label. I checked the plausible ones in
testable areas, and almost all are already claimed:

| Issue                                                      | Why skipped                                                                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| #36535 dns.resolve empty host                              | Open PR #36539                                                                                                      |
| #36533 execFileSync numeric tty fd                         | Open PR #36582                                                                                                      |
| #36519 process.debugPort permission check                  | PRs #36905, #36595                                                                                                  |
| #36445 fmt editorconfig end_of_line                        | PR #36520                                                                                                           |
| #36399 transpile .ts→.js import extensions                  | PR #36403                                                                                                           |
| #36540 compile --bundle --keep-names                       | PR #36684, plus a contributor claim                                                                                 |
| #35172 URL `///test`                                       | Root cause is in the upstream `url` crate (servo/rust-url#1134). The servo maintainer asked for no LLM PRs.         |
| #34793 --no-remote vendor                                  | PR #34827                                                                                                           |
| #31878 DecompressionStream types                           | PR #35216                                                                                                           |
| #31831 fmt JSDoc hard breaks                               | Upstream dprint (dprint-plugin-typescript#800)                                                                      |
| #30887 task --cached-only                                  | PR #36815                                                                                                           |
| #30652 stdin cancel leaves op_read                         | PR #36350                                                                                                           |
| #30219 afterEach masks failure                             | Lives in std/testing; maintainer fork commit                                                                        |
| #30218 sqlite enableLoadExtension                          | Contributor claim                                                                                                   |
| #29829 DecompressionStream deflate                         | PR #32872, maintainer commit                                                                                        |
| #26900 test --doc TS6192                                   | PR #35218                                                                                                           |
| #26554 HTTP Host header                                    | Overlaps open maintainer PR #36481 ("reject invalid Host authorities")                                              |
| #25235 resource leak false positive                        | Maintainer says the leak is genuine (not a bug)                                                                     |
| #24197 octokit Content-Length                              | Maintainer fork commit                                                                                              |
| #23881 `(0) => 1` REPL                                     | swc/upstream                                                                                                        |
| #22956 prompt() newline on EOF                             | PR #32013 and several fork commits                                                                                  |
| #22807 Deno.errors.NotFound type                           | Fork commit and contributor claim                                                                                   |
| #22328 REPL regex                                          | PR #36914                                                                                                           |
| #22171 test --parallel output                              | Not confirmed by a maintainer                                                                                       |
| #21565 prompt/alert/confirm styles                         | Already resolved in current code (no styles anywhere)                                                               |
| #20451 Deno.inspect error colors                           | Maintainer PR #21743 closed unmerged (design question)                                                              |
| #20274 fetch Content-Length streaming                      | PR #35109, assigned to divybot                                                                                      |
| #16246 fetch reader.closed                                 | Assigned to divybot, maintainer commit                                                                              |
| #14664 REPL lone surrogate                                 | Blocked upstream (serde-rs/json#890)                                                                                |
| #14340, #13084                                             | Assigned to divybot (2026)                                                                                          |
| #12221 worker unhandled rejection                          | Large behavior change, old PRs                                                                                      |
| #10988 fetch body chunk validation                         | PR #35927, assigned to crowlbot                                                                                     |
| Type:Bug (not labeled) #32858, #33386, #32583, #35173      | Each has PRs or a crowlbot assignment                                                                               |

## How to reproduce locally (Linux)

```bash
git fetch https://github.com/b84330808/deno claude/eager-turing-s0as7p
git checkout FETCH_HEAD
git submodule update --init tests/util/std
cargo build --bin deno
cargo test -p unit_tests --test unit -- command_test
./tools/format.js && ./tools/lint.js --js
```
