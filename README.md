# /vibecheck
<human-written>
A Claude Code skill that quizzes you on code you’re about to ship.

You run `/vibecheck` on your branch or commit, and Claude asks you questions about the code. It detects gaps in your understanding and points you to the files and teammates that can fill them. When you demonstrate solid comprehension, it drafts documentation and commit messages based on your own words.

## Why this exists

LLMs can generate code faster than developers can internalize it. A developer can accept a suggestion, watch tests pass, and ship something they couldn’t explain under pressure. This is a new kind of technical debt that will bite us in production and future velocity.

Vibecheck exists to help mitigate that loss of familiarity that can come with Claude writing code.

When you struggle to explain something, Claude tells you which file to read and which teammate to talk to, then it stops. This is the most important design decision in the tool. Claude cannot understand the code for you, but it can do a very good job pretending that it can. This is why the skill expressly forbids the LLM from filling in gaps for the user.

This skill will surface the gaps, the job to fill the gaps is yours, and the exercise will be rewarded with better understanding, code, and documentation.

</human-written>
## Install

```bash
mkdir -p ~/.claude/skills/vibecheck && curl -o ~/.claude/skills/vibecheck/SKILL.md https://raw.githubusercontent.com/erichsend/vibecheck.skill/refs/heads/main/SKILL.md
```

## Usage

```
/vibecheck          # quiz me on my current branch
/vibecheck --retro  # quiz me on existing code I need to own
```

**`/vibecheck`** runs against your current branch diff. Claude auto-calibrates depth — a one-file config change gets two questions, a new auth flow across six files gets eight. You don’t choose the rigor. The change does.

**`/vibecheck --retro`** is for code you didn’t write but need to understand. Point it at a file, directory, or commit range. Same conversational style, oriented around “can you own this” instead of “do you know what you just did.” Good for onboarding, post-incident review, or inheriting a module.

## What a session looks like

Claude asks one question at a time and adapts based on your answers. Questions are specific to your code — actual function names, file paths, edge cases — not abstract software engineering trivia.

When your explanation doesn’t match the code, Claude names the mismatch and asks you to resolve it. Either your mental model is wrong or the code is wrong. Both are worth catching before you ship.

When you demonstrate solid understanding, your own words become the raw material for outputs: doc comments, commit messages, and a lightweight ownership record. Claude structures what you said. It doesn’t invent anything.

When a change touches infrastructure, failure handling, or external dependencies, Claude also asks how you’ll know if it breaks in production. It doesn’t suggest monitoring strategies — it can’t see your dashboards or runbooks. It just makes sure you’ve thought about it.

## What it produces

All outputs come from what you said during the session. Nothing is interpolated.

- **Comprehension map** — where you were solid, where there were gaps
- **Gap report** — what was unclear and where you were pointed to look (checked on subsequent runs)
- **Suggested improvements** — things you identified during conversation (rename that function, add that test)
- **Draft documentation** — your plain-language explanations, formatted as comments or README sections
- **Commit message draft** — built from your words, not from what Claude thinks the code does
- **Ownership record** — who confirmed understanding of what, and when

## What it won’t do

- **Explain code to you.** The cardinal rule. Claude asks, detects, and points. Never answers its own questions.
- **Score or grade you.** No pass/fail. Comprehension isn’t binary and scores create gaming.
- **Block your merge.** It’s a mirror, not a gate. You should want to run it.
- **Suggest monitoring or rollback strategies.** It asks whether you have a plan. It can’t know what the right plan is.
- **Simulate a teammate’s review.** If you want a colleague’s perspective, talk to the colleague.
- **Let you pick “easy mode.”** Depth is determined by the change, not your patience.

## Uninstall

```bash
rm -rf ~/.claude/skills/vibecheck
```

## Design document

See <vibecheck-design.md> for the full design philosophy, question archetypes, conversation dynamics, and success criteria.

## License

MIT
