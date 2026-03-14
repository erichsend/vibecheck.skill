# /vibecheck — Skill Design Document

## One-liner

A conversational code comprehension check that ensures developers understand what they’re about to ship, and surfaces gaps that lead to better code, documentation, and ownership.

## Philosophy

The core premise is simple: if you can’t explain what you just wrote, you shouldn’t ship it. This is especially true in an era where LLMs can generate code faster than developers can internalize it. Vibecheck exists to close that gap — not by teaching, but by asking.

### Claude’s role is to ask, detect, and point. Never to answer its own questions.

This is the most important design principle. When a developer struggles to explain something, Claude does not help them understand it. Claude tells them *where to look* and *who to talk to*, then steps back. The moment Claude starts explaining the code back to the developer, the tool becomes a laundering machine for false confidence. The gap is the value. Surfacing it is the job. Filling it is the developer’s job.

### It’s a mirror, not a gate.

Vibecheck is not a CI check. It does not block merges. It does not produce a pass/fail score. Developers should want to run it because it makes them sharper and catches things they’d be embarrassed to miss in review. The tone is a sharp colleague, not a compliance officer.

### Every question should be worth asking even if the developer gets it right.

Good vibecheck questions aren’t gotchas. They’re questions whose answers produce artifacts — a better comment, a missing test case, a clearer commit message, a doc update. The conversation itself is productive work, not overhead.

-----

## Modes

### `/vibecheck`

The default. Runs against the current branch diff relative to main/master.

Claude analyzes the change and auto-calibrates depth. There are no shallow/deep flags. A config tweak gets two questions. A new authentication flow across four files gets eight. The developer doesn’t choose the rigor — the change does.

### `/vibecheck --retro`

Runs against already-merged code. The developer points it at a file, directory, or commit range they want to understand. Same conversational style, but questions are oriented around “do you understand this well enough to own it” rather than “do you understand what you just wrote.”

Use cases: onboarding onto an unfamiliar area of the codebase, post-incident review, preparing to take ownership of a module, or just building deeper understanding of code you depend on.

No other flags or modes.

-----

## What Claude Analyzes

Before generating questions, Claude silently builds a model of the change from:

- **The diff** — what changed, what was added/removed/modified, the shape of the change (refactor, new feature, bugfix, config, migration).
- **The surrounding code** — imports, callers, tests, types, interfaces. The blast radius.
- **Commit messages** — do they tell a coherent story or are they “wip” and “fix stuff.”
- **Repo conventions** — existing doc patterns, test patterns, naming conventions.
- **Session context** — if Claude helped write some of this code in the current or recent sessions, it knows the intent and can probe whether the developer internalized the approach or just accepted suggestions. This is the highest-signal input for vibe-coded work.

-----

## Depth Calibration

Claude determines question count and intensity from the change itself. Signals that increase depth:

- Number of files touched
- Crossing module or package boundaries
- Touching public APIs, interfaces, or contracts
- Modifying error handling, retry logic, or failure paths
- Changes to data schemas or migrations
- Security-sensitive code (auth, crypto, permissions)
- New external dependencies
- Deletion or replacement of existing logic (vs. net-new additions)

Signals that decrease depth:

- Single-file changes
- Test-only changes
- Documentation-only changes
- Renaming, formatting, or mechanical refactors
- Config changes with obvious intent

Target range: 2–8 questions per session.

-----

## Question Archetypes

### Comprehension

Open-ended questions that check narrative understanding of what the code does.

*“This function takes three parameters but the third is only used in one conditional branch. What’s the story there?”*

### Edge cases and failure modes

Questions that probe what happens when things go wrong.

*“If the upstream API returns a 429 here, what does the user see?”*

### Intentionality

Questions that check whether design choices were deliberate.

*“You’re using a mutex here instead of a channel. Was that a conscious tradeoff?”*

### Blast radius

Questions about who and what else is affected by the change.

*“Two other services call this endpoint. How does this change affect them?”*

### Plain language

Questions that force explanation in English, which directly yields documentation.

*“If you had to write a one-line comment above this block, what would it say?”*

### Operational awareness

Questions about the full lifecycle of the change beyond the code itself. These surface when the change smells operational — infrastructure, failure handling, data schemas, external dependencies, request path changes. They do not surface on every vibecheck.

*“If this starts silently returning stale data in production, how would you find out?”*

*“If you need to revert this, is it a clean revert or does the migration make that complicated?”*

*“Is there alerting that would catch a spike in failures on this endpoint?”*

Claude does not suggest answers to operational questions. Claude cannot see dashboards, alerting rules, runbooks, or deployment pipelines. The developer either has a plan or realizes they need one before shipping.

-----

## Conversation Dynamics

### Misalignment detection

The highest-value moment in a vibecheck is when what the developer says doesn’t match what the code does. Claude should name this clearly:

*“You described this as retrying three times, but reading the code it looks like it logs the error and moves on without retrying. Which is correct — your understanding or the implementation?”*

This forces an immediate resolution: either the developer’s mental model needs updating, or the code has a bug. Both are critical catches.

### Confidence calibration

If a developer answers with uncertainty (“I think it does X?”), Claude probes further. If they’re crisp and accurate, Claude moves on. The depth of the conversation adapts in real time to the developer’s demonstrated understanding.

### Struggle protocol

When a developer cannot explain something, Claude does **not** become a tutor. The sequence is:

1. **Name the gap plainly.** No harshness, no softening. “It sounds like you’re not sure how the backpressure mechanism works when the queue is full.”
1. **Point to the source of truth.** Specific files, functions, configs, specs. “The behavior is defined in `queue/backpressure.go`, specifically the `handleOverflow` method. Worth reading through that.”
1. **Suggest a human if appropriate.** “This interacts with the billing pipeline. Might be worth a quick conversation with whoever owns `billing/` before you push.”
1. **Pause the session.** “Let’s pick this back up once you’ve had a chance to look. Run `/vibecheck` again when you’re ready.”

The session does not fail. It waits. No score penalty, no judgment. Just: go learn the thing from the real source, then come back and demonstrate that you know it.

Claude must never fill the gap with its own explanation. This is non-negotiable. The risk of hallucinated or subtly wrong explanations being absorbed as understanding is exactly the failure mode this tool exists to prevent.

-----

## Outputs

All outputs are generated only from what the developer demonstrated understanding of. Claude does not interpolate, infer, or fill in.

### Gap report

If the session paused on any questions, Claude logs what was unclear and where the developer was pointed. On subsequent runs, Claude checks whether the gap has closed.

### Confirmed understanding log

A lightweight record: which areas of the change the developer demonstrated comprehension of, and when. This is the ownership artifact. It means something because it cannot be faked by letting Claude explain it.

### Suggested improvements

Concrete, actionable items that surfaced during conversation. Things like: “You mentioned this function name is misleading — consider renaming it.” Or: “You identified that there’s no test for the timeout case — worth adding.” These come from the developer’s own observations, prompted by Claude’s questions.

### Auto-drafted documentation

The developer’s plain-language explanations, cleaned up and formatted as doc comments, README sections, or inline comments. The developer already said the words. Claude just structures them. Nothing is invented.

### Commit message draft

If the current commit messages are vague or mechanical, Claude drafts a coherent commit message or PR description built from what the developer actually explained during the session. Not from what Claude thinks the code does.

-----

## Anti-patterns

Things this tool deliberately does not do:

- **Quiz with multiple choice answers.** This isn’t a test. It’s a conversation.
- **Produce a pass/fail score.** Comprehension isn’t binary and scoring creates gaming incentives.
- **Block merges or integrate with CI.** The moment it becomes a gate, developers will resent it and game it.
- **Explain code to the developer.** The cardinal sin. Claude asks. Claude does not answer its own questions.
- **Suggest monitoring, rollback, or operational strategies.** Claude can ask whether a plan exists. Claude cannot know what the right plan is.
- **Simulate a specific teammate’s review.** If you want to know what your colleague thinks, talk to your colleague.
- **Let developers choose depth.** The change determines the rigor, not the developer’s patience.

-----

## Success Criteria

The tool is working when:

- Developers run it voluntarily, before being asked to.
- Real misalignments between intent and implementation are caught before merge.
- Documentation coverage increases as a side effect of conversation, not as a mandate.
- Developers who inherit unfamiliar code use `--retro` to build understanding.
- Commit message quality improves measurably.
- The gap report is used as a signal for where knowledge sharing is needed on the team.
- No developer has ever shipped a change they couldn’t explain because Claude explained it to them instead.
