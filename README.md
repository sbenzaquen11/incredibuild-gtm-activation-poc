# Incredibuild GTM Activation Engine
This repository is the technical proof of concept for the Incredibuild GTM Engineering Challenge. It accompanies a slide deck that covers the funnel diagnosis, the thesis, and the full architecture. This repo is the working MVP referenced in that deck.

Because the N8N workflow requires other tools to function, the engine itself is only a mockup, of what it could be. Additional testing would be needed in order to ensure the right data quality output. 

## What it does (Theoretically)
The engine is a conversion system that reduces funnel leakage. The full system serves three jobs, Acquisition, Activation, and Expansion, as three configurations of one parameterized engine. This repo implements the Activation configuration, the mid-funnel job: turning users who signed up but stalled before a first successful build into activated users. It takes a behavioral "stalled user" event, classifies the user, selects and personalizes outreach with AI, and hands the result to HubSpot to send.

## Architecture
The design splits responsibilities deliberately.

*HubSpot is the brain*. It is the system of record and the campaign orchestration layer: lifecycle stages, lead scoring, segmentation, the email sends, the sequences. HubSpot owns funnel state and decides when to call the engine.
*n8n is the muscle*. It is a service HubSpot calls for the work HubSpot does not do well (today) : enrichment, AI classification, AI message generation, and connectors to other tools.
*PostHog feeds the behavioral triggers to Hubspot*

This keeps campaign logic in a tool the marketing team can see and own, and uses n8n only for the specialized steps. The orchestration layer is platform-agnostic; it can run on the company's existing Workato or on n8n.

## Order of operations
 
1. **Webhook trigger.** HubSpot calls the engine on a stalled-user event.
2. **Account-state gate.** Is this an existing customer?
3. **Existing customer** goes to a Slack alert for the CS or account owner, plus a
   HubSpot task. A stalled user at a current customer is an expansion signal for
   CS, not an acquisition nudge. The flow ends.
4. **Not a customer** continues into the engine:
   - **Enrichment via Clay.** The contact is pushed to Clay, and the flow waits
     for the enriched data to return.
   - **Write the enriched data back to HubSpot.**
   - **Persona classifier, AI.** Resolves the free-text job title into `hands_on`,
     `decision_maker`, or `other`.
   - **Relevance gate.** If persona is `other`, the user is not a relevant
     activation target. The flow ends, no message.
   - **Stall routing.** A Switch routes by stall stage, and a branch-prep step
     sets a human-readable stall reason.
   - **Message-writer agent, AI.** Reads the messaging library, selects the base
     message that best fits the user, and personalizes it into a first email. It
     can decline to write if nothing fits.
   - **Push the email to a HubSpot property**, which triggers the HubSpot sequence.
5. **HubSpot sends the sequence.** This is HubSpot-side, not n8n.

## The AI components
**Persona classifier.** Job titles are free text and vary widely. Mapping them to
a hands-on versus decision-maker split is a genuine fuzzy task, so it uses an LLM.
Routing by stall stage, by contrast, stays a deterministic Switch, because stall
stage is a clean controlled field. AI is used where the data is ambiguous, rules
where it is clean. Prompt: `prompts/persona-classifier-prompt.md`.
 
**Message-writer agent.** It receives the full messaging library and the user
context, selects the best-fit base message, and personalizes it. It does not
invent the angle or the offer; those come from the library. It reports which
library entry it used, so every email is traceable. It can return
`generate: false` when no message fits. Prompt: `prompts/message-writer-prompt.md`.
 
**Messaging library.** The base messages product marketing owns. The engine
selects from and personalizes these; it never writes copy from scratch. This is
the role boundary in practice: marketing owns the messages, the engine owns the
system. Library: `message-library/message-library.json`.


## Key assumptions
 
- The funnel counts net-new contacts; existing-customer events are routed to CS,
  not nurtured.
- A free or trial product motion exists, so self-serve signup and setup are real
  funnel stages.
- No live data access, so events are mocked and external systems are stubbed.
- Granular product event instrumentation, such as setup sub-steps and build
  outcomes, is assumed. In production some of this is an instrumentation task for
  the product team.

  # Repository contents
 
```
README.md
workflow/activation-engine.json      the n8n workflow export
prompts/persona-classifier-prompt.md
prompts/message-writer-prompt.md
message-library/message-library.json
mock-data/                           three sample stalled-user events
docs/canvas.png                      the workflow canvas
```
