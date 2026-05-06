# Re-Grounding Workflow

Loaded by `ce-replan-beta`'s Phase 2. Re-grounding is the core move that distinguishes a replan from an in-place plan edit. The original plan and PR are **evidence to interrogate**, not authoritative framing to inherit.

## Four steps

### 1. Read artifacts in this order

Read in this order specifically. Order matters because it determines which framing the agent absorbs first — and the goal is to absorb the user's discussion before the original plan's distillation of it.

1. **PR review threads, comments, and review bodies** — the user's actual back-and-forth, where the original plan's assumptions got tested. Pay attention to where reviewers pushed back, where the user changed their mind, and where confusion accumulated.
2. **Recent docs in `docs/brainstorms/`** — any new requirements docs that postdate the original plan. These often *are* the new learning.
3. **Prior conversation context** — what the user said in the current session that prompted the replan.
4. **Original plan doc** — last. Read for what was decided, but treat the framing as a hypothesis under review, not as a starting point.

When dispatching subagents to read PR data, pass the path to a saved JSON dump (e.g., from `scripts/fetch-pr-context.sh`) rather than inlining the content. See `docs/solutions/skill-design/pass-paths-not-content-to-subagents-2026-03-26.md` for the rationale.

### 2. Re-derive the problem frame from the user's words

Compose a fresh problem-frame paragraph using the **user's language from the discussion**, not from the original plan's distilled prose. Specifically:

- Quote (or close-paraphrase) what the user said about the pain in PR threads or recent conversation.
- Identify the moment of pain: what triggered the replan? A specific reviewer comment, a code-reading discovery, a brainstorm doc?
- State what the user thought before the new learnings, and what they think now. The **delta** is the reason the replan exists.

The original plan's `## Problem Frame` may overlap with this — that's fine. The discipline is to derive the new problem frame independently, then notice where it agrees or disagrees with the original.

### 3. Re-question every original requirement

For each requirement in the original plan, mark one of:

- **`[unchanged]`** — the requirement still holds as stated. Default for requirements where no learning contradicts them. No further reasoning needed.
- **`[revise]`** — the requirement still has a real intent behind it, but the learnings reshape what it should say. Capture the original wording, the revision, and the specific learning that drove the change.
- **`[discard]`** — the requirement was tied to an approach the replan abandons, and there is no underlying intent worth carrying forward. Capture the original wording and the reason it's gone.

This step satisfies the no-silent-inheritance rule: every original requirement is touched, and the disposition is visible in the output doc. Default to `[unchanged]` is intentional — the agent does not invent reasons to revise. But never silently drop a requirement, even one the new approach makes obviously irrelevant: surface it as `[discard]` with a one-line rationale.

If the original plan had no explicit requirements section (older or terse plans), derive the implicit requirements first — what behavior was the plan trying to deliver — then mark each.

After requirements, do the same pass on:

- **Scope boundaries** — is anything previously excluded now in scope? Is anything previously in scope now out?
- **Key technical decisions** — which decisions still hold, which need revisiting?

### 4. Compose a three-bucket synthesis for the user

Mirror the synthesis pattern used by `ce-plan` and `ce-brainstorm`. Format:

```
Based on the PR, original plan, and new learnings, here's the scope I'm proposing for the replan:

[1-3 line prose summary — what's changing and why, in plain language. Forward-looking.]

**Stated** (carried forward from the original PR + plan + learnings):
- [item]

**Inferred** (gaps I filled with assumptions — flag anything I got wrong):
- [item]

**Out of scope** (deliberately excluded — including things the original plan kept that the replan drops):
- [item]
```

Use prose for the user response (no `AskUserQuestion` menu) — option sets bias the answer. The user confirms, revises, or redirects. **In pipeline mode**, skip the prompt and route Inferred bets to the output doc's `## Assumptions` section.

## Anti-patterns to avoid

- **Diff-against-the-plan thinking.** Do not treat re-grounding as "what changed in the plan." The plan is downstream of the user story; re-derive the user story first, then let the plan changes fall out.
- **Preserving the original plan's framing language.** If the new problem-frame paragraph reads like a paraphrase of the original, the inheritance has already happened. Use the user's discussion language instead.
- **Critique-mode.** This is not a review of the original PR's quality. It is a re-derivation in light of new information. The original work was correct given what was known then; the replan exists because the world changed.
- **Skipping requirements.** "All the original requirements obviously still hold" is a tell that step 3 was not actually performed. Each requirement gets an explicit `[unchanged]` / `[revise]` / `[discard]` mark, not a blanket assumption.
- **Brainstorm-from-zero.** The opposite failure: ignoring the original artifacts entirely and re-deriving the user story from scratch. The point is to interrogate the original, not throw it away. The PR's working code, designs, and IDs are real evidence of what the user wanted; the replan picks up what's still good.

## Worked example: the brief-view scenario

A long-running PR (`origin/brief-view`) introduced a new sidecar table for tracking briefed-state transitions. The original plan called for migrations, a new ERD entry, and several new mailer hooks. After several rounds of review, the user discovered an existing saved-view system could be repurposed without any new tables.

A naive replan would say: "drop the migration, use the saved view, keep everything else." That is the in-place edit failure.

A re-grounding pass instead asks:

1. **Artifacts read** — PR threads (where reviewers questioned the sidecar's necessity), the new brainstorm noting the saved-view alternative, original plan with the sidecar requirements, current conversation.
2. **Re-derived problem frame** — the user wanted to track when emails became "briefed" so list counts and chips updated correctly. They thought a sidecar was needed because they didn't yet know the saved-view system existed.
3. **Requirement re-questioning** — "create Briefed sidecar table" → `[discard]` (no underlying intent — the goal was tracking, not the table). "Update list counts when briefed transitions" → `[unchanged]` (still required). "Add mailer hook on briefing" → `[revise]` (still needed but now hooks into saved-view machinery, not the sidecar).
4. **Synthesis** — Stated: tracking briefed transitions, list count updates, mailer behavior. Inferred: saved-view repurposing is the right approach for v1 (the user said "I think this works" but did not test it under load). Out: the sidecar table, the ERD update, the migration.

The output plan then names the sidecar work as the discarded approach, cherry-picks the UI components and IDs that were correct independent of the storage choice, references `origin/brief-view` as the superseded PR, and instructs the user to start fresh from `main`.

The worked-example level of detail is illustrative — actual replans will have shorter and longer cases.
