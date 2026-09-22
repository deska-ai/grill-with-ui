# Browser interaction protocol

The shared `server.mjs` owns validation and persistence. Use its existing commands and
fields; the calling workflow's domain model belongs in its own artifacts, not new state keys.
`state.json` is agent-owned, `events.jsonl` is page-owned. Read state when needed; write it
only through `patch`. Never edit either file to simulate a browser action.

## Publish a round

Run from the caller's project directory, using the runtime path resolved in `SKILL.md`:

```sh
node "$RUNTIME/server.mjs" patch --session "<session>" <<'GRILL_PATCH'
{
  "agent": {"status": "waiting"},
  "questions": [{
    "id": "q1", "round": 1, "deps": [],
    "title": "Which audience?",
    "body": "Who should the first release serve?",
    "options": [{"k": "A", "text": "Existing customers"}, {"k": "B", "text": "Everyone"}],
    "rec": {"option": "A", "why": "Existing customers can validate the workflow first."}
  }]
}
GRILL_PATCH
```

Use stable, case-sensitive question IDs. `deps` contains question IDs supplied by the caller;
`round` preserves its grouping. For a free-text question use `options: []` and
`rec: {text: "<recommendation>", why: "<reason>"}`. Optional `durable` records the caller's
classification, not a classification invented by this layer. `terms` entries use
`{term, def, avoid: []}` when the caller supplies vocabulary.

`patch --file <path>` accepts the same JSON from a file. It validates before saving; a
rejected patch exits nonzero and leaves state unchanged. Correct the reported violation.

## Merge semantics

- `questions` merges entries by `id`; a new entry needs `round`, `title`, and `rec`.
  Existing fields are replaced whole, including `answer`, `options`, `rec`, and `explore`.
- `agent` and `visual` merge one level. `terms` entries are replaced by matching `term`.
- Question `thread`, visual `thread`, and `visual.queued` append. Send only new entries.
  `visual.queued` entries are text bullets, for example `["feedback: show the failure path"]`.
- `null` deletes a key. Other top-level values are replaced whole, including `finished`.
- The server stamps omitted timestamps. Use the event's `at` for the user's thread message;
  do not manufacture timestamps for agent status, replies, drawing, or finishing.

## Handling actions

An event has `{type: "send", seq, at, session, actions: [...]}`. `pending` and `wait` return
the same event format as the server's live output. Use events from the selected session.

1. Patch `agent.status: "working"`.
2. Process actions in their recorded order, collecting changes:

| Action | State change and caller handoff |
| --- | --- |
| `answer` | Copy its `kind` (`accept`, `option`, or `text`) and any supplied `option` or `text` into the named question's `answer`; accepted recommendations can carry an option too. Set `status: "answered"`, `updated: false`. Pass the answer to the caller. |
| `thread` | Append `{who: "user", text, at}` and the caller's `{who: "agent", text}` reply to the question thread. Discussion alone does not answer the question. |
| `defer` | Set the question's `status: "deferred"`; pass the deferral to the caller. |
| `reopen` | Set `status: "reopened"`, `answer: null`; let the caller reconsider dependent questions. |
| `explore` | Obtain option trade-offs from the caller and set `explore: {rows: [{option, pros: [], cons: []}]}` in option order. If its recommendation changes, patch `rec` and `updated: true`. |
| `visualize` | Request the caller's visual work through [the visual lifecycle](visual.md). Both Visualize and Regenerate send this action. |
| `visual-feedback` | Append the user's message and the caller's reply to `visual.thread`, then request visual work. If feedback contradicts an answered question, reopen it and quote the feedback in its question thread. Update its recommendation as appropriate, but preserve the recorded answer until the user answers again. |
| `finish` | Apply the caller's Finish requirements after processing preceding actions. See `SKILL.md`. |

3. Apply the caller's next questions and recommendation changes. Changed recommendations
   on open questions set `updated: true`. Preserve unrelated questions, terms, and visual state.
4. If the discussion affects an existing visual, set `visual.stale: true`. Ordinary answers
   do not regenerate it or change `version`, `at`, or `note`.
5. Publish changes from steps 2–4 in **one patch**, with
   `agent: {status: "waiting", handled: <last processed seq>}`. Do not publish a new round
   separately from its acknowledgement; the page needs both to clear staged sends.
6. Return to listening. An idle wait timeout is not a completed interaction.

For a backlog of sends, process all in order before selecting the next round and publishing
the combined patch. The acknowledgement is the last processed event, never merely the last
event observed. Explore, visualize, visual feedback, and finish may arrive immediately
when their controls are clicked, without a separate Send action.
