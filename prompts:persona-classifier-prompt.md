# Persona Classifier Prompt

This prompt classifies a stalled user's job role into a persona category. The
Activation engine uses the result two ways: to decide whether the user is a
relevant activation target at all, and, if so, to frame the message. Hands-on
users get technical help, decision-makers get an offer to route the work to
their team.

In the repo this file lives at `prompts/persona-classifier-prompt.md`.

## Why this is an AI step and routing is not

Routing by funnel stage stays a deterministic Switch node, because `stall_stage`
is a clean controlled field with exact values. There is nothing to infer.

Persona is different. Job titles are free text and vary widely: "Staff Engineer,"
"Build & Release Lead," "VP of R&D," "Head of Developer Experience." Mapping that
text to a hands-on versus decision-maker split, and spotting roles that are not a
relevant target at all, is a genuine fuzzy classification task where an LLM beats
brittle string matching. AI is used where the data is ambiguous, rules where it
is clean.

## How it is used in the flow

1. The node runs after enrichment and before the message step.
2. It receives the user's job role.
3. It returns a persona label as JSON.
4. A small parser node merges `persona` back onto the event item.
5. A relevance gate, an IF node, checks the persona. If it is `other`, the user
   is not a relevant activation target and the flow stops, no message is generated.
6. For `hands_on` and `decision_maker`, the flow continues to the message step.

Set the node temperature low, around 0.1. This is classification, not creativity.

## System prompt

```
You classify a software tool user into a persona category, used both to decide
whether the user is a relevant target for activation messaging and to route the
message that gets written.

Categories:
- hands_on: writes code or directly runs builds and CI. Software developers,
  engineers at any level including senior, staff, and principal, build engineers,
  DevOps, release engineers, SREs. These users do technical setup themselves.
- decision_maker: leads or manages an engineering function and is unlikely to do
  hands-on setup personally. Engineering managers, team leads, directors, VPs of
  Engineering or R&D, CTOs, heads of platform or developer experience.
- other: not a relevant target for activation messaging. The role has no clear
  hands-on or engineering-leadership function. Procurement, finance, marketing,
  sales, recruiters, students, or any role that does not fit the two above. When
  the persona is other, the flow will not generate a message.

Rules:
- Decide from the job role text.
- If the role is clearly an engineering role but the seniority is ambiguous,
  choose hands_on. Most users at this stage do the setup themselves.
- Choose other only when the role is clearly outside engineering. Do not use it
  as a catch-all for unusual engineering titles.
- Output strict JSON only. No preamble, no markdown, no code fences.

Output schema:
{"persona": "hands_on | decision_maker | other", "reasoning": "one short sentence"}
```

## Input prompt (template)

```
Job role: {{ job_role }}

Classify the persona as JSON.
```

## Output examples

```
{"persona": "decision_maker", "reasoning": "An Engineering Manager leads a team and typically delegates hands-on setup."}
```

```
{"persona": "other", "reasoning": "A Procurement Specialist has no engineering function and is not an activation target."}
```

## Design notes

- `other` is the relevance gate. It is the signal that the flow should not spend
  a message on this user. The downstream IF node acts on it.
- The `reasoning` field makes the classification auditable. It is for debugging
  and review, not shown to the user.
- The ambiguity rule defaults unusual engineering titles to `hands_on`, so a
  genuine target is never dropped just because the title is unfamiliar. `other`
  is reserved for roles that are clearly not engineering.
