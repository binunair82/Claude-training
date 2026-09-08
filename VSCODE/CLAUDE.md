# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

Not one project — a workspace holding **two unrelated, self-contained deliverables**:

| Path | What it is |
| --- | --- |
| `index.html` | UOB IT PMO Kanban board — single-file demo/training web app |
| `SSL/` | Certificate Injection Utility — `cert-inject.sh` (bash + keytool) plus `app.py`, a web UI front end for it |

`Training/` and `VSCODE/` are empty. There is no git repo here, so there is no history to consult — treat each file as the whole record.

The two deliverables share one deliberate constraint: **zero third-party dependencies**. No npm, no pip, no CDN. Both are meant to be copied somewhere and run as-is. Reaching for React, Flask, or any package manager breaks the point of both.

## Environment traps on this machine

These are verified, and each one silently blocks an obvious approach:

- **No `node`, `npm`, or `deno`.** To parse-check or unit-test JavaScript, use JavaScriptCore instead:
  `/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Helpers/jsc`
- **No JDK.** `/usr/bin/java` and `/usr/bin/keytool` exist but are macOS stubs that error with "Unable to locate a Java Runtime". `cert-inject.sh` therefore **cannot be run end-to-end here** — only its argument parsing and `--help` path.
- **`/bin/bash` is 3.2.** `cert-inject.sh:213` uses `${path,,}` (bash 4+ lowercase expansion), which fails at runtime with `bad substitution` under macOS bash. `bash -n` still passes, so a syntax check will *not* catch it. The script targets Linux servers; test it there.

---

## `index.html` — UOB IT PMO Kanban board

### Hard constraints (breaking any of these breaks the deliverable)

- **One file.** All markup, exactly one `<style>` block, exactly one `<script>` block.
- **Vanilla only.** No framework, bundler, or build step.
- **No external resources.** No CDN, no web fonts, no image files — system font stack, inline SVG and Unicode glyphs. The single permitted outbound URL is the FormSubmit endpoint.
- **No persistence, deliberately.** No `localStorage`, `sessionStorage`, IndexedDB, or cookies. A refresh resetting the board to the 8 seeded tasks is *intended behaviour*, and the header says so. Do not "fix" this.
- Must work opened directly from `file://` by double-clicking.

### Theming

The board ships light and dark palettes, reachable two ways:

- **the OS setting**, via `prefers-color-scheme`, which is the default; and
- **the header's Theme button**, which cycles System → Light → Dark and writes
  `data-theme` onto `<html>` to override the OS in either direction.

The button's choice is held in `state.theme` **in memory only**. The
no-persistence rule above bars the storage it would need, so the theme resets
on refresh — the same way the eight seeded tasks do, which the header already
says. Writing `data-theme` is the **third documented write outside
`renderBoard()`**, alongside the `.drag-over` class and toast insertion.

Section 12 of the stylesheet restates the palette under two selectors, because
CSS cannot union a media query with an attribute selector:
`@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }` and
`:root[data-theme="dark"]`. **Those two token lists must be kept in sync.**
They are tokens only — no component rule is duplicated — so anything new must
reach for a token rather than a literal colour, or it will be correct in one
theme and wrong in the other. `color-scheme` is set per theme so the native
date picker, selects and scrollbars follow along.

Both palettes are contrast-verified: body text ≥ 13:1 dark / ≥ 16:1 light, muted
text ≥ 5.4:1, every priority pill ≥ 4.8:1, and focus rings plus input borders
≥ 3:1 against whichever surface sits behind them. Re-measure after changing any
colour — the light palette previously shipped three pairs below 4.5:1
(`.meta-key`, `.pill-low`, `.pill-high`) and an input border at 1.77:1.

Two traps when measuring, both of which produced wrong answers here first time:

- **Anything on the header gradient.** `background-color` is transparent there,
  so a script that walks up looking for an opaque ancestor sails past and
  compares against the page background instead. The Theme button shipped at
  1.81:1 for exactly this reason — `.btn-ghost`'s muted grey on dark blue —
  and the automated sweep called it a pass. Composite the translucent fill over
  both gradient stops by hand, or sample rendered pixels.
- **Playwright screenshots** of this page hang on the stability wait unless you
  pass `animations: 'disabled'`.

### Architecture

A single `state` object is the source of truth, and **all card markup is produced by `renderBoard()`**. Nothing outside the render path mutates card contents.

The consequence worth internalising: transient UI states that would normally live in the DOM live in `state` instead — `confirmingDelete` (the inline "Delete? Yes / No" row, used because native `confirm()` is banned) and `openMoveMenu` (the keyboard-accessible drag-and-drop fallback). Toggling either is just a re-render.

Because every card is rebuilt on each render, **all board events are delegated** from the `#board` container (`data-action` attributes dispatched in `handleBoardClick`) — attaching listeners to individual cards would orphan them. For the same reason, focus would be lost on every re-render, so `state.refocus` holds a `data-focus` key that `restoreFocus()` reapplies after each render. Preserve that handshake when adding controls.

Two documented exceptions to "no DOM mutation outside render": the `.drag-over` highlight class on a column, and toast insertion into its own live region.

Every user-supplied string goes through `escapeHtml()` before reaching `innerHTML`, including quoted attribute values.

### Security posture

- A **Content-Security-Policy meta tag** sits in `<head>`. `'unsafe-inline'` is
  unavoidable (one inline `<style>`, one inline `<script>`), but `default-src
  'none'` plus `connect-src https://formsubmit.co` means an injected `<img>`,
  `<iframe>` or `<script src>` loads nothing and cannot beacon data out. Verified
  in-browser: FormSubmit allowed, every other host blocked. Adding any new
  outbound host means editing the policy too.
- `frame-ancestors` is deliberately **absent** — meta-tag CSP ignores it and
  GitHub Pages cannot set headers, so clickjacking cover would need a real host.
  Do not add it and imply protection that is not there.
- Outbound notifications are capped in memory at `SEND_LIMIT` per
  `SEND_WINDOW_MS` (5/minute). This is an abuse limiter, **not** a security
  control: it resets on refresh and cannot stop anyone POSTing to the endpoint
  directly. It exists so the public demo cannot be turned into a mail bomb.
- `escapeHtml()` covers `& < > " '` and every interpolation into `innerHTML`
  passes through it, with all attribute values quoted. This is the primary XSS
  defence; the CSP is the backstop.

**Open issue — the notification address is public.** `FORMSUBMIT_ENDPOINT`
contains a real mailbox in plaintext, committed to a public repo and served on
a public Pages site, where it is harvestable and can be POSTed to by anyone.
See the README for the two options (dedicated alias, or drop the feature).

### FormSubmit

`FORMSUBMIT_ENDPOINT` near the top of the script is the **only** place the notification address appears; change it there and nowhere else. Requires a one-time activation — the first submission emails a confirmation link to that address, and nothing is delivered until it is clicked.

Submission is **optimistic**: the card is added and rendered before the network call, which runs in parallel wrapped in `try`/`catch`. A FormSubmit failure must never break the board — it downgrades to a warning toast. Note that over `file://` the JSON content-type triggers a CORS preflight from a `null` origin that some browsers refuse, so this failure path is routinely exercised in normal use; serving over HTTP avoids it.

### Verifying changes

No test framework exists. This recipe (used to validate the original build) runs the **real** script headless — extract the `<script>` block, define a small DOM stub, then `load()` it and assert against the actual functions:

```bash
SP=/tmp/kanban-check && mkdir -p $SP
awk '/^<script>$/{f=1;next} /^<\/script>$/{f=0} f' index.html > $SP/app.js
JSC=/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Helpers/jsc
# parse check
echo 'var s=readFile("'$SP'/app.js"); try{new Function(s);print("PARSE OK")}catch(e){print("PARSE ERROR: "+e)}' > $SP/c.js && $JSC $SP/c.js
```

`escapeHtml`, `applyFilters`, `validateForm`, `isOverdue`, `renderCard`, `addTask`/`moveTask`/`deleteTask` are all pure or state-only and testable this way. Stub `document.getElementById` to return a fake element with `innerHTML`, `value`, `classList`, `appendChild`, `addEventListener`, `focus` and `querySelectorAll`.

Two greps worth re-running after any edit:

```bash
grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie' index.html   # comments only
grep -noE 'https?://[a-zA-Z0-9./-]+' index.html | sort -u                      # formsubmit.co only
```

Also cross-check that every `getElementById("…")` in the script has a matching `id="…"` in the markup — the two halves of the file drift easily.

Interaction behaviour (drag and drop, focus trap, responsive stacking below 768px) needs a browser: `open index.html`.

---

## `SSL/` — Certificate Injection Utility

### Division of responsibility

`cert-inject.sh` is the **single source of truth** for the keytool logic. `app.py` is a Python-standard-library web UI that does not reimplement any of it — it builds an argv, shells out via `subprocess.run`, and renders the captured output. Keep both files together; `app.py` refuses to start if the script is missing or non-executable.

When changing behaviour, change the script; `app.py` only needs touching if a new *flag* has to be exposed in the form.

### Contracts between the two

- **Passwords never appear in argv.** `app.py` passes them as the `KEYSTORE_PASS` / `TRUSTSTORE_PASS` environment variables, and strips them from the re-rendered form so they are never echoed back. The script reads flags → env vars → hidden interactive prompt, in that order. Preserve this ordering and the no-echo rule.
- **Exit codes are an API** the UI surfaces: `0` success · `1` bad usage/input · `2` missing tool or file · `3` keystore import failed · `4` truststore import failed. The 3-vs-4 split matters — a `4` means the keystore was already modified and only the truststore needs restoring from backup, and the error message says so.
- The script backs up both stores to `cert_injection_backups/` and appends to `cert_injection.log` **relative to its working directory**, which `app.py` controls via `--workdir`.

### Running

```bash
./SSL/cert-inject.sh --help                    # works on macOS; real runs need Linux + a JDK
./SSL/cert-inject.sh -k ks.jks -t ts.jks -c server.crt -p ./certs -a myserver --check
python3 SSL/app.py                             # http://127.0.0.1:8443
UI_USERNAME=admin UI_PASSWORD='…' python3 SSL/app.py --host 0.0.0.0
```

`--check` validates inputs and makes no changes — the safe way to exercise the path without a keystore to damage.

`app.py` binds to localhost and leaves HTTP Basic Auth **off** unless `UI_USERNAME` and `UI_PASSWORD` are both set; it warns on stderr if bound to a non-localhost address without them. Anything beyond localhost should have auth or a reverse proxy in front, since the page accepts store passwords.
