# GSDL Model Tiers

GSDL routes each pipeline step to a **capability tier**, never to a named model. The tiers are
provider-neutral, so the same pipeline runs on Anthropic, OpenAI, Google, or anything else that
offers more than one model size.

## The three tiers

| Tier | Used for | Why |
|---|---|---|
| **LARGE** | Clarifying intent, writing `SPEC.md`, slicing the work, synthesising the decisions document | Open-ended judgment. Mistakes here propagate through every downstream step, so this is where the capability is worth paying for |
| **MEDIUM** | Implementing an already-approved plan, running gates, opening the PR and triaging bot findings | Execution against a reviewed brief. The hard thinking already happened and was checked by a human |
| **SMALL** | Tracker reads and writes, filling templates from fetched content | Resolve an ID, call one API, report the result. There is no reasoning here to pay for, and these calls are frequent |

## Mapping to a provider

`## Models` → `provider` in [`config.md`](./config.md) selects the mapping:

| Tier | Anthropic | OpenAI | Google | Any other provider |
|---|---|---|---|---|
| **LARGE** | Opus | The current flagship reasoning model | Gemini Pro | The largest / most capable model available |
| **MEDIUM** | Sonnet | The current mid-tier general model | Gemini Flash | The mid-sized workhorse model |
| **SMALL** | Haiku | The current small / cheap model | Gemini Flash-Lite | The smallest model that can follow a template reliably |

**Resolve model families, not version strings.** Use whatever the provider currently offers in that
family — do not pin a dated model ID in a skill. To pin one anyway, set it explicitly under
`## Models` → `large` / `medium` / `small` in the config, and that value wins for that tier.

If `provider` is unset, infer it from the session's own model and use that provider's mapping.

## Routing rules

1. **Never run a step below its tier.** Silently downgrading is the failure this section exists to
   prevent. Upgrading is allowed but never required — running a SMALL step inline because the
   session is already on a LARGE model is fine.
2. **MEDIUM and SMALL steps are subagents**, spawned with an explicit model override. State the
   tier and the resolved model in the subagent's prompt too, so the choice is recorded even where
   the override isn't honoured by the agent tool.
3. **LARGE steps are interactive.** If the session is already on a LARGE model, run them **inline**
   — the Q&A is direct, with no relay. Otherwise spawn a LARGE subagent and use the Question Relay
   protocol in the `gsdl` orchestrator.
4. **Subagents never talk to the user.** Anything interactive comes back to the orchestrator as a
   returned blocker or question.
5. **All tracker I/O runs on SMALL** via `gsdl-tracker-sync`. Never spend a LARGE or MEDIUM model
   on an API round-trip.

## When model overrides aren't available

Some agent tools run every subagent on the session model, and some sessions have only one model
available at all. That is a supported configuration:

- Run every step on the session model, in the order and with the checkpoints the pipeline defines.
- **Still state the intended tier** in each subagent's prompt and in `progress.md`, so the record
  shows what the step was meant to run on.
- Tell the user once, at the start, that tier routing is unavailable in this setup — do not repeat
  it at every step, and never stop the pipeline over it.

The tiers are a cost and quality optimisation. The checkpoints are the part that must not be
skipped.
