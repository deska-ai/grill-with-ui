# Visual lifecycle

The caller supplies visual meaning, the renderer or worker instructions, and any structured
source data. This layer displays the resulting `<session>/visual.html` using the existing
`visual` fields. Set `kind` to the supported `prototype` or `diagram` value that fits the
output. Producing or viewing an artifact does not accept its contents.

## HTML handoff

Produce one self-contained HTML file with inline CSS and JavaScript and embedded data.
The page displays it in an iframe with `sandbox="allow-scripts"`; it cannot read the session,
parent page, or local files. Avoid external assets and requests. Validate the rendered file
before publishing its version; the caller owns domain correctness and distinguishes evidence,
proposals, and unresolved choices in the display.

Read the current interview state and the caller's source artifacts for each request. Include
the whole current discussion, not only the most recent answer. Keep domain models out of
the page's transport state. Do not load the original `visual-brief.md` unless the caller
explicitly chooses its prototype/diagram authoring method.

## Request and completion

Retain the existing upstream `drawing`, `queued`, `stale`, and `version` lifecycle:

1. On Visualize, Regenerate, explicit visual feedback, or required Finish reconciliation,
   note the current `visual.html` modification time, if any, and start the caller's visual
   work. Use its worker/context policy; this presentation layer does not choose a model or
   an architectural planning skill. If no visual method is supplied, ask for it and
   acknowledge the event without claiming a draw is running.
2. Acknowledge a dispatched request in the send's combined patch. For the first draw use
   `visual: {kind, version: 0, thread: [], stale: false, drawing: {seq}}`.
   For a redraw use `visual: {stale: false, drawing: {seq}}`, preserving the previous visual
   and version. Return to listening while the worker operates.
3. During a draw, continue handling answers and discussions. Relevant new answers set
   `stale: true`; the worker did not see them. Further explicit draw requests append to
   `visual.queued` as text bullets, for example `queued: ["regenerate with current answers"]`.
   Do not dispatch a second writer of the
   same file. Several triggers in one send form one request with their combined changes.
4. On completion, verify that the HTML exists, changed since dispatch, and passed the
   caller's rendering check. Patch `visual: {version: <previous + 1>, note: "<what changed>",
   drawing: null}`. Leave `stale` untouched so newer answers remain visible as pending work.
   A completion notice is not an acknowledgement of any new user event.
5. If requests are queued, dispatch the next draw with the current record and queued feedback,
   clear `queued` with `null`, and set its `drawing: {seq: <last handled seq>}` in the same
   completion patch. Apply step 2's `stale: false` for this new dispatch; subsequent answers
   can mark it stale again. Completion alone never clears stale. Apply the caller's
   worker/context policy again for each dispatch.
6. On failure or unchanged output, do not increment the version. For a first draw clear
   `visual` with `null` and explain the failure in the page `note`. For a failed redraw,
   clear `drawing`, retain the displayed version, set `stale: true`, and append an agent
   message to `visual.thread`. Report any queued requests that still need handling.

The page reloads the iframe on a version increment; writing HTML without publishing a
new version leaves the old visual displayed. Never increment the version without a new
validated artifact. Ordinary interview turns only mark a visual stale.

If the caller uses a synchronous renderer, finish that rendering and publish the result
in the send's combined acknowledgement patch. Do not claim a background worker exists.
If it requires a fresh worker but the harness cannot provide one, report that missing
capability instead of silently switching methods. Do not add scheduling or retry machinery.

## Finishing with a visual

Before export, reconcile all answered questions and visual feedback, including changes
received during a draw. Continue listening until the caller's final artifact is ready.
Copy the reconciled HTML beside its document, then publish both paths in `finished` as
described in `SKILL.md`. If reconciliation fails, keep completion pending and explain why.
