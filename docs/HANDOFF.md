# Grill-Me UI — Handoff

Read this first, then `docs/SPEC.md`. The spec says **what** to build and is agreed. This file fills the gaps an implementer would otherwise guess at.

Everything below marked **(proposed)** was not part of the grilling session. Follow it unless the owner says otherwise. Where it conflicts with the spec, the spec wins. Where a choice comes up that neither file covers, pick the simplest option that keeps the spec's behaviour, and record it in the log at the end of this file.

`docs/spec.html` is a rendered copy of the spec for people. Don't edit it, and don't treat it as a second source.

## Starting point

- The repo contains only `docs/`. No code exists yet.
- Development machine: Python 3.12.3, Claude Code 2.1.304, logged in. Linux.
- No PyPI packages, ever. That includes tests: use `unittest` from the standard library.
- Work in the build order from the spec. Make one commit per feature. Get the thinnest slice (features 1, 2, 4 and 5) running end to end before starting on 6.

## Decisions the spec leaves open

### 1. Where the grilling skill comes from

The source is the `grilling` skill from Matt Pocock's skills plugin (MIT licence), version 1.2.3. On the development machine it is at:

```
~/.claude/plugins/cache/mattpocock/mattpocock-skills/1.2.3/skills/productivity/grilling/SKILL.md
```

Copy it to `grill/.claude/skills/grilling/SKILL.md`, keep its frontmatter `name: grilling`, and put a line under the frontmatter crediting the source and the MIT licence. Then make these changes:

| Original behaviour | Adapted behaviour |
|---|---|
| Asks the whole frontier per round, numbered `Q1`, `Q2`… | Asks **exactly one** question per turn: the single most important open question on the frontier. Keep the `❓ **<title>**: <body>` format, but without numbering. |
| `➡️` recommendation after each question | Unchanged. Every question ends with one `➡️ <recommended answer>` line. |
| Dispatches sub-agents to look up facts | Remove. The session has no tools. Facts come only from `project.md` and the user. If a fact is missing, ask the user. |
| Design tree is open-ended | Every section heading of the template is a branch the grill must visit. A branch is done when that section could be filled from the user's answers. |
| Ends when the frontier is empty and the user confirms | Same end condition, then output the filled template between a line `<<<FILLED_TEMPLATE` and a line `FILLED_TEMPLATE>>>`, with nothing else between the markers but the document. |
| (none) | When told to fill the template early, fill it straight away, and write `_not yet discussed_` under any section without material. |

### 2. `project.md` and `template.md`

The owner hasn't supplied real ones. **(proposed)** Write small placeholder files so the app can be tested, and say clearly in the commit message that they are placeholders:

- `grill/project.md`: a half-page description of a made-up project, for example "a shared office plant-watering rota".
- `grill/template.md`: a short design-doc template with 4–5 `##` sections, such as *Goal*, *Users*, *Scope*, *Risks* and *Open questions*, each with one line of guidance in italics.

### 3. The hidden prompts (proposed)

Kickoff, sent as the first stdin message of a new session:

```
/grilling

You are grilling the user about the project below. The grill is not done until every section of the template below can be filled in from the user's answers. Ask your first question now.

<project>
{contents of project.md}
</project>

<template>
{contents of template.md}
</template>
```

Finish, sent when the user presses **Finish & fill template**:

```
The user has pressed "Finish & fill template". Stop grilling. Fill in the template now from everything discussed so far. Under any section with no material, write "_not yet discussed_". Output the document between a line <<<FILLED_TEMPLATE and a line FILLED_TEMPLATE>>>.
```

Neither prompt is ever shown in the UI or written to the transcript.

### 4. Turning stdout into chat entries (proposed)

- One turn produces **one** assistant bubble. Collect the `text` content blocks of every `assistant` event until the turn's `result` event, join them with blank lines, and process that text when the `result` arrives.
- Ignore other content block types (thinking, tool use) and other event types.
- If the text contains a complete marker block, take the part between the markers, strip whitespace, save it to `data/outputs/<user_id>.md`, and log a `document` entry. Remove the marker block from the bubble text. If any text is left, log it as an assistant `message` **before** the `document`. If none is left, log only the `document`.
- Only use the first marker block if there are several. If the opening marker appears with no closing one, leave the text in the bubble as it is.
- `ts` is an ISO-8601 UTC string, for example `2026-10-20T09:15:03Z`.

### 5. Server structure (proposed)

The spec diagram says "stdlib, asyncio". That label isn't a decision. Use `http.server.ThreadingHTTPServer`:

- Each SSE connection is a handler thread that blocks on its own `queue.Queue`.
- Each `claude` process gets one reader thread for stdout and one for stderr. The stderr thread keeps the last few lines to use in error cards.
- A single `threading.Lock` per `GrillSession` protects `busy`, the deadline, the attached queues and the process handle. A global lock protects the registry and the live-process count.
- A timer thread checks turn deadlines and idle processes every few seconds.

With a cap of 5 processes, threads are plenty and avoid writing an HTTP parser by hand.

### 6. Edge cases

- **Server busy:** a browser `EventSource` can't read the body of a `503`. **(proposed)** Accept the SSE connection, send one `error` event `{text: "Server busy, try again shortly", can_resume: false, kind: "busy"}`, then close. When the page gets `kind: "busy"`, it closes its `EventSource` (so it doesn't reconnect forever) and shows a *Retry* button. Keep `503` for any non-SSE request that would need a process.
- **Keep-alive:** send an SSE comment line (`: ping`) every 15 seconds so proxies don't drop idle streams. If a write fails, detach that tab.
- **Reconnects:** the browser reconnects `EventSource` on its own, and each new connection replays the full transcript. The page clears the chat before a replay. **(proposed)** The server sends one `replay_done` event after the replay. Until it arrives, the page doesn't scroll or animate.
- **Status on attach:** after the replay, send the current `status` so a tab opened mid-turn shows the countdown straight away.
- **Error before `system/init`:** there's no `session_id` to resume, so the next attempt starts fresh with the kickoff. The transcript is empty in that case anyway.
- **Detecting a failed resume:** the process was started with `--resume`, and it exits or reports `is_error` before finishing its first turn. Then send `error` with `can_resume: false`, and offer *New grill* in the card. Don't delete anything until the user confirms.
- **Process exits while idle,** for example killed from outside: drop it from the registry. No error card unless a turn was running.
- **Cookie:** `user_id` is `secrets.token_urlsafe(16)`. Reject any cookie value that doesn't match `^[A-Za-z0-9_-]{10,64}$` and issue a new one, because the value is used in file paths.
- **Writing to stdin:** one JSON object per line, followed by `flush()`. Set `busy` and the deadline **before** writing.

## Testing (proposed)

Never call the real `claude` in automated tests. It's slow, costs money and needs a login.

- Add a `--claude-cmd` option to `server.py` (default `claude`), and pass the spec's flags after it.
- Write `tests/fake_claude.py`: it reads stdin lines and prints canned stream-json (`system/init` with a fixed `session_id`, then an `assistant` event and a `result` for every user line). Trigger words in the user text switch on error modes, for example `FAKE_ERROR` (a `result` with `is_error`), `FAKE_HANG` (never answers), `FAKE_EXIT` (exits), `FAKE_FILL` (replies with a marker block). Make it accept `--resume <id>` and fail if `<id>` is unknown.
- Tests start the server on a free port with temporary `data/` and grill folders, then drive it with `urllib` and a small SSE reader.
- Pass the timeouts and cap as command-line options or environment variables so tests can set short values. Keep the spec's numbers as the defaults.

Run the tests with `python3 -m unittest discover -s tests`.

## Done checks per feature

Each check runs against the fake unless it says **(real)**.

1. `GET /` returns the page and sets the cookie once. A second request with the cookie doesn't set a new one.
2. The first SSE attach spawns one process and sends the kickoff. The kickoff never shows in the transcript. **(real)** The first message is a single grilling question ending in `➡️`.
3. **(real)** Three turns in a row each ask exactly one question. Answering everything leads to a marker block.
4. Two tabs see the same entries in the same order. A third tab opened later replays them.
5. Markdown renders. `<script>` in a message is shown as text and not run. Deleting `vendor/` falls back to plain text.
6. A `POST` during a turn returns `409`. Every tab shows the same countdown. `FAKE_HANG` produces an error card after the (shortened) timeout.
7. Restarting the server and reattaching resumes with the same `session_id` and no kickoff.
8. `FAKE_FILL` and `/finish` both produce a card. `/download` returns the file. A second fill overwrites it.
9. `FAKE_ERROR` and `FAKE_EXIT` produce red cards that aren't in the transcript after a reload. The next message resumes.
10. *New grill* wipes the three kinds of state and kicks off again. The idle reaper kills a process with no tabs and keeps its files. A sixth user gets the busy event.

## Repo workflow

- `app/data/` is runtime state. It's in `.gitignore`.
- Vendor the two JavaScript files with the `curl` commands in the spec and commit them.
- Commit messages: the feature number and name, for example `Feature 4: transcript and fan-out`.

## Implementer's log

Record here any decision you make that neither this file nor the spec covers. Give one line per decision, with the reason.

- _(empty)_
