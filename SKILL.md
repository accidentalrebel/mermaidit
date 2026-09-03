---
name: mermaidit
description: |
  Turn what is currently on the table - a plan, an architecture, a flow, a lifecycle - into a real
  Mermaid diagram written to a .mmd file, and hand back a link that opens it in the mermaid-canvas
  editor, where the human edits it visually and Claude reads it back as source. Use when the user
  invokes /mermaidit, says "diagram this", "draw this out", "turn that plan into a diagram", or
  asks to revise a diagram from earlier in the session. NOT for a read-only page mixing diagrams
  with findings, annotated code or wireframes - that is visualize-this. NOT for compressing a reply
  to its bottom line - that is tldr.
---

# Mermaidit

Draw what is being discussed, put it in a file both sides can edit, hand back a link.

The diagram is the output. It lives in the browser, not in the terminal.

## 1. Work out what to draw

Bare `/mermaidit` draws whatever is currently being discussed — the plan just proposed, the
architecture just worked out. That is the common case and needs no argument.

An argument narrows it rather than replacing it: `/mermaidit the auth flow` means draw the auth
flow out of the current discussion. If the argument resolves to a file path, read that file and
draw its contents instead.

If the conversation has nothing diagram-shaped in it and no argument was given, say so and ask what
to draw. Do not invent a subject.

## 2. Get the facts before drawing

A diagram that invents structure is worse than no diagram, because the structure makes it look
verified.

- **The subject exists on disk** (a codebase, a config, a running system): read the files and trace
  the real path before drawing. Not from memory of the conversation.
- **The subject does not exist yet** (a plan, a proposed design): the conversation is the source.
  Say so in the handback line so the reader knows it is a proposal, not a survey.

## 3. Pick the type, and say what it costs

Pick from the content: interactions over time are a sequence diagram, structure is a flowchart, a
lifecycle is a state diagram, data shapes are ER. Picking wrong is the main way one of these fails
to help.

But the editor's click-to-edit layer only works on flowcharts:

| Type | Editing in the canvas |
|---|---|
| flowchart | Click any label to rename it |
| sequence, state, ER, class, mindmap | Source pane only |

Measured, not assumed: state, ER and class diagrams produce elements the editor targets, but no
label resolves back to a source span, so all of them go inert. Sequence diagrams produce no
targets at all.

So choose the best type, and say which mode the user is getting. Do not silently trade away visual
editing.

## 4. Write labels that do not break

Measured against the vendored bundle. Do not reason about these; they are counter-intuitive.

| In a label | Bare? |
|---|---|
| `<br/>` | Yes, and it is the only way to get a line break |
| `<` or `>` in the middle | Yes |
| `[` `]` `(` `)` `{` `}` `\|` | **No.** Terminates the shape early — quote the whole label |
| A leading `>` | Quote it. `A[>x]` reads back as `x`, so the `>` is lost on the next edit |
| `"` | Write it as `#quot;` |
| `<` immediately followed by a letter | **No.** Read as an HTML tag and it silently eats the rest of the label. Write `&lt;` |

That last one is the live footgun: `A[<arrow]` renders as an empty box. `A[x <- y]` is fine,
because the `<` is followed by a space.

## 5. Size it for comprehension

Past roughly fifteen nodes a diagram usually stops being clearer than the prose it replaced. Below
that, just draw it. At or past it, ask the user whether they want comprehension (group and
abstract, offer to expand a part) or fidelity (draw everything), and follow the answer.

## 6. Where the file goes

Scratch by default: `${MERMAID_CANVAS_ROOT:-$HOME/development}/mermaid-scratch/<topic-slug>.mmd`.
That default is a guess, not a query against the running server — writing here must not require
the canvas to be up, since it might not be (step 8 handles that as its own state). Step 8's
handback check is what actually proves this landed somewhere the link can reach; if the server's
real root ever diverges from this default, that is where it surfaces, not here. If the name is
taken, use `<topic-slug>-2.mmd` — never overwrite, since the existing one may carry hand edits.

It has to be that directory, not a dotdir and not a symlink elsewhere: the canvas scan skips
directories starting with `.` and does not follow symlinked directories, so anything else is
invisible in the file list. Scratch is swept at 7 days by mtime.

Only promote a diagram out of scratch when the user asks. Then propose a destination — usually
`<the relevant repo>/docs/` — say where it is going, and move it once they agree.

Promotion does not change how a diagram edits. A promoted sequence diagram is still source-pane
only, permanently, in whatever repo it lands in. Say which mode it has when proposing the move, so
nobody discovers it later.

## 7. Check it renders before handing it over

Check for the script itself, not merely the checkout — a checkout can have one without the other.
Keep the existence check in its own statement, so that only the linter's own exit code is ever
read as a verdict on the diagram:

```bash
canvas="${MERMAID_CANVAS_DIR:-$HOME/development/mermaid-canvas}"
if [ -f "$canvas/scripts/lint-mmd.mjs" ]; then
  node "$canvas/scripts/lint-mmd.mjs" <file>
fi
```

A missing script means **skip this step**. It never means the diagram is broken. Chaining the two
with `&&` would collapse those into the same non-zero exit, and you would sit there fixing a
diagram that was fine all along.

The path is a default, not a fact. If the checkout lives somewhere else, `MERMAID_CANVAS_DIR`
points at it. When the script is not found, say which path you looked at — otherwise "skipped the
render check" reads as "you have no canvas" when the real answer is "your canvas is somewhere I
did not look", and those want different fixes.

When the linter does run, non-zero means it does not parse: read the error, fix the source,
re-check. Do not hand over a diagram that failed; cap the retries at three and say what is wrong
if it will not converge.

If it prints `SKIP`, the dev dependencies are missing — `npm install` in that checkout.

With no script and no canvas at all, the label rules above are the fallback, and they are what
lets this skill work on a machine that has never heard of mermaid-canvas.

## 8. Hand it back

Derive the URL, never hardcode a hostname:

```bash
listing=$(serve.sh --list 2>/dev/null)
base=$(printf '%s\n' "$listing" | grep -oE '^https://[^ ]+' | head -1)
printf '%s\n' "$listing" | grep -q ' /canvas ' && exposed=yes || exposed=no
```

Match the URL, do not take the first word of the first line. When nothing is currently served that
line is not a URL at all, and blindly slicing it hands back a link built from a stray word.

With `$base` non-empty and `exposed=yes`, ask the server for its own root rather than assuming
`~/development` — `MERMAID_CANVAS_ROOT` is a shell variable this skill has no reason to have set,
it only reads back as the right answer today because a systemd unit happens to default it the same
way:

```bash
root=$(curl -s "$base/?op=list" | grep -oE '"root": *"[^"]*"' | sed -E 's/.*"([^"]*)"$/\1/')
rel="${file#$root/}"   # $file is the diagram's full path from step 6, never hand-typed with a
                        # literal ~ - an unexpanded tilde makes this substitution a silent no-op
```

A file written per step 6 to `<root>/mermaid-scratch/<slug>.mmd` must produce
`rel=mermaid-scratch/<slug>.mmd`. Hand-deriving that once already dropped the `mermaid-scratch/`
segment, which silently 404s as "no such diagram" since the server has no basename-fallback
search. `$root` here is a different thing than `MERMAID_CANVAS_DIR` in step 7 (that one points at
the `mermaid-canvas` checkout itself, for running its linter; this one is the server's scan root,
a level above the checkout) — do not conflate the two.

**Prove the link before handing it over, the same way a browser would reach it:**

```bash
check=$(curl -s "$base/?op=load&path=$rel")
printf '%s' "$check" | grep -q '"content"' && verified=yes || verified=no
```

Only a `verified=yes` link gets handed back. A wrong root, a mis-stripped path, or a typo in `$file`
all produce a dead link that looks identical to a working one until clicked — this is the check
that would have caught the original bug before the user ever saw it, instead of after. If
`verified=no`, do not hand back the URL: say the link did not resolve, show what `$check` returned,
and give the raw `$file` path instead so the diagram is still reachable.

With `$base` empty, or `exposed=no`, or the verification above failing: there is no link to give —
say the canvas is not exposed (or that the built link did not verify), give the file path instead,
and mention `serve.sh http://127.0.0.1:8898 canvas --permanent` as the fix for the exposure case.
Never hand over a URL you did not actually construct from a match and confirm resolves.

Then one line in the terminal, carrying three things: the diagram type, why that type, and which
editing mode it gives. For example: *"Sequence diagram, since this is a message exchange over time
— labels edit in the source pane, not by clicking."* Nothing else. Do not print the mermaid source;
that is the wall of text the diagram replaces.

**If the canvas is not running:** `systemctl --user is-active mermaid-canvas.service`. If it is
installed but stopped, offer to start it and proceed once the user agrees. If `mermaid-canvas` is
not checked out at all, write the file, print the source, and say the canvas is unavailable — the
source still renders in any markdown viewer.

**If `TELEGRAM_BRIDGE_ORIGIN=1`**, the turn came from the user's phone and the terminal is not a
channel back to them. Send the link with `telegram-sender`.

## 9. Revising

Never regenerate a diagram wholesale. Read the current file, change only what is being revised, and
leave the rest byte-identical — the user may have spent time arranging it in the browser, and those
edits are in the same file.

Default to the last diagram this session wrote or opened; an explicit path wins. Name the file
before changing it, so a wrong guess is visible rather than silent.

## Gotchas

- **A quoted label stays quoted.** The editor's `encodeLabel` returns early on an already-quoted
  span, so quotes already in a file are never removed by an edit. If a label looks over-quoted,
  rewrite that line in the source rather than expecting a rename to clean it.
- **The scan is depth-bounded at 6 and skips `node_modules`, `.git`, virtualenvs.** A diagram
  written deeper than that loads by deep link but never appears in the dropdown.
- **Anyone on the tailnet can overwrite any `.mmd` under the root.** Auth is tailnet membership.
  Do not draw anything into a scratch diagram that should not be readable by every device on it.
