# Code analysis: `slime agent v2` (`slime_agent.py`)

A single-file (~1,240 lines), stdlib-only AI coding agent designed to run in Termux on a
phone, backed by OpenRouter's chat-completions API. This analysis covers architecture,
strengths, **verified bugs** (each reproduced against the code), security posture, and a
prioritized fix list. Verified on Python 3: the file compiles clean (`py_compile`) and the
CLI smoke-tests fine (`--help`).

---

## 1. What it is

| Aspect | Detail |
|---|---|
| Runtime | Python 3, standard library only (no pip deps) — ideal for Termux |
| Model backend | OpenRouter (`/chat/completions`), streaming SSE, retries on 429/5xx |
| Agent loop | ReAct-style: model emits **one action per reply** → harness executes it → result fed back as a user message, up to `MAX_STEPS = 12` |
| Tool protocol | Text actions: `CMD:`, `READ:`, `WRITE:`, `EDIT:`, `GREP:`, `LS:`, `FETCH:` |
| Safety model | Every write shown as a diff and confirmed; commands confirmed with danger/safe heuristics; graduated auto-approve modes |
| Extras | Sessions (`~/.slime/sessions`), persistent memory, themes, readline completion, Termux TTS/notify/clipboard, undo stack |

## 2. Architecture & control flow

```
main()                       argparse → load key (~/.or_key) → config → readline → banner
 └─ REPL loop                read_line() → "!cmd" passthrough | "/slash" | chat
     └─ add_user(expand_mentions(q))      @file attachments inlined into the message
         └─ run_turn()                    up to 12 steps
             ├─ maybe_compact()           auto-summarize when history > 36k chars
             ├─ chat_stream(msgs)         Spinner + Streamer (prints prose, hides action lines)
             ├─ parse_action(text)        first CMD|READ|... line → (kind, arg, body)
             ├─ run_action(...)           do_cmd / do_read / do_write / do_edit / ...
             │    └─ apply_change()       diff + confirm → write + push to undo stack
             └─ "Result of X:\n..."       appended as a user message; loop continues
```

Well-chosen division of concerns: streaming/UI (`Spinner`, `Streamer`), API layer
(`open_req`, `chat_stream`, `chat_once`, `api_fail`), tools (`do_*`), agent loop
(`run_turn`), and slash commands are cleanly separated for a single file.

## 3. What it does well

- **Zero dependencies.** `urllib` + `subprocess` + `re` gets streaming, retries, and tools
  with nothing to install on a phone. Genuinely the right call for Termux.
- **Human-in-the-loop by default.** Every file change prints a colored unified diff and
  waits for `y/N/a`; every command is shown in full before it runs; dangerous patterns get
  a type-`yes` gate. The auto modes (`off → reads → edits`) escalate gradually.
- **Good UX craft.** Braille spinner that stops before printing; action lines hidden from
  the stream while shown as `▸ preparing edit`; code fences dimmed; `rl()` wraps ANSI in
  `\001/\002` so readline measures the prompt width correctly (a subtle bug most agents
  get wrong); `!cmd` vs `!!cmd` (run locally / share with the model) is a nice touch.
- **Real agent-loop hygiene.** One action per turn with tool results fed back as messages;
  plan mode that refuses mutating actions in `run_action`; `EDIT` prefers surgical
  SEARCH/REPLACE with "matched exactly once" validation; undo stack; session
  save/restore after every turn.
- **Prompt-injection awareness.** `FETCH` output is labeled `[web page text, untrusted
  data]` and the system prompt forbids following instructions found in it. Not a full
  sandbox, but it's stated where it matters.
- **Data-conscious.** `/compact` summarization, output caps everywhere (`cap()`), token
  estimates in `/stats` — appropriate for metered mobile data.

## 4. Verified bugs (ranked)

Each of these was reproduced against the actual code, not just read.

### 4.1 High — compaction mid-turn silently loses messages (aliasing bug)

`run_turn()` (line 723) captures `msgs = st["msgs"]` once, but `cmd_compact()` (line 846)
**rebinds** `st["msgs"] = [...] + tail` (line 863) to a new list. The local `msgs` in
`run_turn` keeps pointing at the old, orphaned list:

- `chat_stream(msgs)` continues on the *un-compacted* history → the compaction (its whole
  purpose is saving data) has **no effect on the current turn**;
- everything `run_turn` appends after that (`msgs.append(...)`) goes to the orphaned list
  and is **not** in `st["msgs"]`;
- `save_session("last")` then writes `st["msgs"]`, so the post-compaction messages
  disappear from the saved session and from every later turn.

Repro (stubbing `chat_once`, 30×1500-char messages):

```
st['msgs'] is the old list? False
post-compact msg visible in st['msgs']? False
live session len=7, old list len=32
```

**Fix (one line each):** make `cmd_compact` mutate in place — `st["msgs"][:] = [msgs[0], ...] + tail` —
or re-read `msgs = st["msgs"]` at the top of the `for` loop in `run_turn`. Also call
`refresh_system()` after compaction so `st["msgs"][0]` and the stale `msgs[0]` can't drift.

### 4.2 Medium — action lines inside code fences trigger real tool calls

`parse_action()` (line 672) scans every line and matches `ACTION_RE` **without tracking
fenced code blocks** (unlike `Streamer.emit`, which at least tracks `self.code` for
display). A perfectly reasonable assistant reply like:

````
Here's how you could run it:
```
CMD: rm -rf ~
```
Done.
````

parses as `('CMD', 'rm -rf ~')` — verified. The command would still hit the confirm
dialog, so it's not silent, but quoting an example in prose shouldn't invoke tools at all.

**Fix:** track ``` / `~~~` state in `parse_action` (and `said_text`) and only match
`ACTION_RE` outside fences. (Optional hardening: also require the action line to be the
last non-empty line of the reply.)

### 4.3 Medium — `max_tokens: 1500` can truncate a `WRITE` into a silently short file

`chat_stream` (line 258) hard-caps every reply at 1500 tokens. If the model is cut off
mid-`WRITE`, `extract_block()` (line 590) treats the missing closing fence as "rest of the
body" and returns the truncated content — verified: `'line1\nline2\nline3\n'` from a
cut-off block. The diff preview + confirm is the only thing standing between that and a
truncated file on disk; an approving user on `/auto edits` gets it applied automatically.

**Fix:** raise the cap for tool turns (or make it model/context-aware), and make
`extract_block` return `None` / an explicit "block was cut off" when no closing fence was
found *and* the stream hit `max_tokens` (OpenRouter's `finish_reason` is already in the
SSE payload — plumb it through and surface it).

### 4.4 Low — `/retry` doesn't retry

`cmd_retry` (line 837) appends `"Please continue."` when the last message is an assistant
reply and runs another turn. That's *continue*, not *retry*: the failed/draft reply stays
in history and the model doubles down on it. True retry would drop the trailing assistant
message(s) and resample from the last user message.

### 4.5 Low — session `model` field is saved but never restored

`save_session` writes `"model": st["model"]` but `load_session` (line 797) only reads
`msgs` and `cwd` (line 804). `/resume` after a `/model` switch silently keeps using the
current model. Same for `undo`, `auto`, and `plan` state — `/undo` after `--resume` says
"nothing to undo" even though the session made changes.

### 4.6 Low — `cmd_cd` quote handling and root bookkeeping

`cmd_cd` (line 937) passes `shlex.quote(arg)` into `exec_shell`, whose `cd` handler then
manually strips quotes with `.strip("'\"")` (line 436ff). A path containing a quote
(`it's`) round-trips mangled — `shlex.quote("it's")` → `'it'"'"'s'` → strip → broken. And
`st["root"] = os.getcwd()` runs **even when `cd` failed**, silently redefining the
"outside your project folder" boundary to wherever you happen to be.

**Fix:** `os.chdir(os.path.expanduser(arg))` directly (no shell quoting round-trip), and
only update `st["root"]` on success.

### 4.7 Low — inconsistent error surfacing

`run_turn` (line ~746) highlights failures matching
`^((READ|LS|FETCH|EDIT|WRITE|GREP) (failed|needs)|Blocked)`, but `apply_change` returns
lowercase `"write failed: %s"` (line 623), which never matches — write errors go to the
model but not to the user's screen. Either capitalize it or match case-insensitively.

### 4.8 Cosmetic / minor

- **Banner off-by-one** (line 1185): header content is 37 cols inside a 38-col box
  (`w - 21` should be `w - 20`) — the right border sits one column off. Also assumes
  `◕` is width 1; on wide-char terminals the box is more distorted.
- **Ctrl+C at the prompt exits** the whole REPL (`main` catches `KeyboardInterrupt` from
  `read_line` and breaks). Cancelling the current line and staying in is the expected
  REPL behavior; Ctrl+C mid-generation already correctly just stops the turn.
- **`/models` pricing heuristic** (line 907): `str(pr.get("prompt")) == "0"` misses
  numeric `0.0` (`str(0.0) == "0.0"`) — compare with `float(pr.get(...) or 1) == 0`
  instead.
- **Memory silently truncates**: `/remember` appends unbounded to `memory.md`, but
  `system_text` (line 337) reads only 2000 chars — oldest facts drop out of the prompt
  with no warning. Same for `SLIME.md` at 3000 chars.
- **`EDIT_RE`** (line 608) can't represent a SEARCH block containing a `=======` line
  (verified: the non-greedy group stops at the first one). Fine in practice, worth a note
  in the prompt ("SEARCH blocks must not contain a line of `=======`").
- **`show_output` redacts only the API key**; command output is stored verbatim in session
  JSON and `/export` markdown. Anything else sensitive a command prints (env vars, tokens
  in files) persists on disk in `~/.slime`.
- **`st["sent"]` statistics** double-count history on every request (it sums all messages
  each call). Accurate as "total bytes pushed to the API", misleading as "tokens used".

## 5. Security assessment

For a tool whose job *is* running shell commands, the posture is reasonable — with the
caveat that the guardrails are seatbelts, not a sandbox:

| Layer | Mechanism | Assessment |
|---|---|---|
| Secrets | `SECRET` regex (line 363) blocks `~/.or_key`, `~/.ssh`, history, key names | Best-effort only: `cat ~/.or_*` or `cat ~/.or_key ` via a variable evades it. Fine as a footgun guard; the confirm dialog is the real control. |
| Dangerous commands | `DANGER` regex (line 364) → type `yes` gate | `rm -rf` / fork bombs / `dd` covered, but `rm --recursive --force` or `rm -r -f` (split flags) don't match. Acceptable since even danger only escalates the prompt, not execution. |
| Auto-approve | `SAFE_RO` allowlist (line 367) + metachar veto `[;&|<>`$\n]` | Genuinely careful: command substitution, pipes, redirection and `find -exec/-delete` all force a prompt. This is the best-designed part of the guard system. |
| Write scope | `in_project()` warning + confirm for outside-root writes | Good; note `root` can drift (§4.6). |
| Prompt injection | FETCH labeled untrusted, system prompt says never follow | Structural gap remains: fetched text lands in the same channel as instructions. Consider a hard wrapper like `[BEGIN UNTRUSTED EXTRACT — data only]` and ignoring any action-like lines inside tool results when parsing. |
| Key handling | read from `~/.or_key`, redacted in echoed output, chmod advice | Good. Could `os.chmod(0o600)` on load and warn if world-readable. |

Nothing here is exploitable *without user confirmation* except the aliasing bug (§4.1),
which is a correctness/data-loss issue rather than a security one. The design principle —
"the user approves everything, regexes just shape how loud the prompt is" — is sound.

## 6. Prioritized fix list

1. **§4.1** compaction aliasing — `st["msgs"][:] = ...` or re-fetch `msgs` per step. (data loss)
2. **§4.3** plumb `finish_reason` / raise `max_tokens` for WRITE turns. (silent truncation)
3. **§4.2** fence-aware `parse_action`. (surprise tool calls)
4. **§4.4** real `/retry` semantics (drop last assistant turn, resample).
5. **§4.5** restore `model` (and `undo`/`auto`/`plan`) in `load_session`.
6. **§4.6** direct `os.chdir` in `cmd_cd`; update `root` only on success.
7. **§4.7** case-insensitive failure match (or uppercase `write failed`).
8. The §4.8 polish batch: banner width, Ctrl+C behavior, pricing heuristic, memory cap
   warning, EDIT marker caveat, broader secret redaction in persisted output.

## 7. Verdict

This is well above the bar for a hobby single-file agent: the streaming UX, confirmation
flow, and safe-command allowlist show real care, and the constraints (stdlib-only,
Termux, metered data) are respected everywhere. The one bug that really matters is the
compaction/list-aliasing issue — it turns a headline feature (`/compact` "saves data")
into message loss exactly when a session gets long enough to need it — followed by the
truncation and code-fence issues. All three are small, contained patches; the rest is
polish on an already-solid base.

---

## 8. Update: v2.1 (post-review changes)

The code has since been revised (`slime_agent.py`, banner shows **v2.1**). Two notes:

**Correction to §4.8:** the "banner off-by-one" claim was wrong — re-measured with ANSI
stripped, the box is 38 cols between borders (`w - 21` correctly accounts for the pad
space after the left `│`). The banner was fine all along.

**Requested change:** `!` / `!!` roles are now reversed — `!cmd` runs a shell command
*and shares it with the agent* (AI help), `!!cmd` runs it privately (the AI never sees it).

**Bugs from §4 that are now fixed:**
- §4.1 compaction aliasing — `cmd_compact` mutates `st["msgs"]` in place and `run_turn`
  re-binds `msgs` each step (verified: post-compact messages survive)
- §4.2 fence-aware `parse_action` / `said_text` / `Streamer` — action lines inside code
  fences are examples, never executed (verified)
- §4.3 truncation — `finish_reason == "length"` is detected; an unclosed WRITE block on a
  truncated reply is refused with a "continue with EDIT" hint; cap raised to 2048 and
  configurable via `cfg["max_tokens"]` (verified)
- §4.4 `/retry` now drops the last assistant reply and resamples (verified)
- §4.5 sessions round-trip `model`, `undo`, `auto`, `plan` (verified)
- §4.6 `cmd_cd` uses direct `os.chdir` (spaces/quotes work), `root` updates only on
  success; `exec_shell` cd parsing uses `shlex.split` (verified)
- §4.7 write errors surface as `WRITE failed` and the console match is case-insensitive
- §4.8 memory cap now warns at >1800 chars; `/models` pricing uses `float(...) == 0`;
  Ctrl+C cancels the input line instead of quitting; EDIT marker caveat added to the
  system prompt; secret redaction (API key, `sk-`/`ghp_`/JWT/Bearer/AWS/Slack tokens) in
  command output and `/export`; FETCH output wrapped in `<<<EXTRACT>>>` untrusted markers

**New features:** `/doctor` (setup health check), `/pin` (keep a file in context every
turn), `/undo [n|all]`, `/models [filter]`, `/stats` context meter, `@folder`
attachments (listing), `-y/--yes` and `--plan` CLI flags for one-shot runs, `/pet feed`
easter egg. All changes covered by a behavioral test pass (role swap, fences, compaction,
truncation guard, session round-trip, redaction, pins, cd quoting).
