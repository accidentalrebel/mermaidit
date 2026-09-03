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

Scratch by default: `~/development/mermaid-scratch/<topic-slug>.mmd`. If that name is taken, use
`<topic-slug>-2.mmd` — never overwrite, since the existing one may carry hand edits.

It has to be that directory, not a dotdir and not a symlink elsewhere: the canvas scan skips
directories starting with `.` and does not follow symlinked directories, so anything else is
invisible in the file list. Scratch is swept at 7 days by mtime.

Only promote a diagram out of scratch when the user asks. Then propose a destination — usually
`<the relevant repo>/docs/` — say where it is going, and move it once they agree.

Promotion does not change how a diagram edits. A promoted sequence diagram is still source-pane
only, permanently, in whatever repo it lands in. Say which mode it has when proposing the move, so
nobody discovers it later.

## 7. Check it renders before handing it over

Check for the script itself, not merely the checkout — a checkout can have one without the other:

```bash
test -f ~/development/mermaid-canvas/scripts/lint-mmd.mjs && \
  node ~/development/mermaid-canvas/scripts/lint-mmd.mjs <file>
```

Non-zero means it does not parse. Read the error, fix the source, re-check. Do not hand over a
diagram that failed; cap the retries at three and say what is wrong if it will not converge.

If the script prints `SKIP`, the dev dependencies are missing — `npm install` in that checkout.
If the script is not there at all, skip this step. The rules above are the fallback, and they are
what lets this skill work on a machine with no canvas at all.

## 8. Hand it back

Derive the URL, never hardcode a hostname:

```bash
base=$(serve.sh --list 2>/dev/null | head -1 | awk '{print $1}')
serve.sh --list 2>/dev/null | grep -q ' /canvas '   # is the canvas exposed?
```

The link is `$base/canvas?path=<path relative to ~/development>`.

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
