---
title: "Pair Programming with AI: Auditing Component Docs Across an 80-Component Library"
excerpt: "I maintain a React component library with 80+ components. The README files had drifted from the actual TypeScript source — wrong props, wrong defaults, missing docs entirely. Here's how I used AI to audit and fix them in one session, and what it actually caught."
description: "I maintain a React component library with 80+ components. The README files had drifted from the actual TypeScript source — wrong props, wrong defaults, missing docs entirely. Here's how I used AI to audit and fix them in one session, and what it actually caught."
categories: pair-programming-with-ai
tags: AI Frontend Architecture React
---

**TL;DR:** I used AI to audit the README files across an 80-component React library. It caught real bugs — wrong defaults, missing props, renamed types — that had gone unnoticed for months. It also introduced 9 new issues that a review pass had to catch. AI is powerful for this kind of work, but verification is non-negotiable.

---

## The Problem Nobody Wants to Fix

I maintain a React component library with over 80 components. It serves multiple brands, multiple teams, and is the foundation of our frontend stack. The components are well-tested and well-typed. The documentation, however, had quietly rotted.

README files listed props that no longer existed. Default values were wrong. Some components had detailed docs that hadn't been updated in over a year. Others — perfectly functional, widely used components — had no README at all. A developer using the library would read the docs, trust what they saw, and write code against it. When the prop name was wrong or the default had changed, they'd hit bugs at runtime with no obvious explanation.

This is the natural entropy of a growing codebase. Nobody introduced these problems on purpose. Someone renames a prop, updates the TypeScript type, writes tests — and forgets to update the README. Multiply that by 80 components and a few years, and you have a documentation layer that actively misleads the people it's supposed to help.

## Why It Never Gets Fixed

The reason is simple: there was no automation. The docs were manually written and manually maintained. There was no CI check to flag when a prop changed but the README didn't. So the drift was silent and gradual.

And here's the thing — nobody is going to volunteer to audit 80 README files against TypeScript source code. It's tedious, cross-cutting, and invisible until something breaks. Before AI tooling, you'd either live with the drift or invest significant time in building automated doc generation. Now there's a third option: point an AI at the gap and let it do the cross-referencing.

That's the practical value I keep coming back to. Not replacing engineers, but making it realistic to tackle the work that was always important but never urgent enough to prioritize.

## The Approach

I used [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) — Anthropic's CLI tool for AI-assisted development. If you haven't used it, think of it as an AI that runs in your terminal, can read and edit your codebase, run commands, and — crucially — dispatch sub-tasks to independent workers.

Claude Code has built-in [skills](https://docs.anthropic.com/en/docs/claude-code/skills) — pre-packaged workflows that guide the AI through structured processes. I used the [Superpowers](https://github.com/anthropics/claude-code-plugins/tree/main/superpowers) skill pack, which includes workflows for brainstorming, planning, parallel execution, and code review. Instead of just saying "fix the docs," the skills guided a deliberate sequence:

1. **Brainstorm** the scope: which components to include, what format to use, what to skip
2. **Write a spec** locking down the requirements: props table format, what to document, what to omit
3. **Create an implementation plan** breaking the work into batches
4. **Execute with parallel agents** — multiple independent AI workers, each handling one component simultaneously
5. **Code review** the output from multiple angles
6. **Fix** everything the review caught

The key insight was treating the AI like a team of focused contributors. Each parallel agent got one clear task: read this component's TypeScript source, compare it to the README, and rewrite the docs to match reality. By batching the components into groups, the entire library was processed in a single session.

The structure is what made it work. Without the spec defining the format, the agents would have produced 80 inconsistent READMEs. Without the plan breaking it into batches, the context would have been too large to manage. The skills provided the guardrails; the AI provided the scale.

## What It Caught

The audit surfaced real bugs that human review had missed for months:

- A text component's default `size` was documented as `'Body Large'` — the actual source code said `'Body'`. Anyone relying on the documented default was working with wrong assumptions.
- Several required props were marked as optional in the README. Developers who omitted them would get TypeScript errors with no explanation from the docs.
- A prop had been renamed in the source (`imageServerProps` → `imageServerKnobs`) but the README still used the old name.
- One component documented a `reduceMotion` prop that the component never actually read — it was defined in the TypeScript interface but the implementation ignored it entirely. Anyone passing that prop was getting silently no-op'd.
- Theme values listed as valid options in the docs (`'fastlane'`) actually mapped to different internal values at runtime (`'ppe'`), meaning any conditional logic based on the documented values would produce dead code.

These aren't hypothetical risks. These are bugs that were silently affecting developers who trusted the documentation.

## What Didn't Work

The AI output wasn't flawless. After all the agents finished their work, I ran a structured code review — a skill that examines diffs from three angles: line-by-line accuracy, removed behavior (did we lose important information?), and cross-file consistency.

The review caught 9 issues the agents had introduced or missed. Required props documented as optional. Wrong theme values carried over. Internal import paths used in usage examples instead of the public package path. One component quietly renders nothing if you don't pass valid children — the old README warned about this, but the agents dropped that note during the rewrite.

The agents' self-review wasn't sufficient. The structured review pass was essential. This is the part that matters most: AI output needs verification, not blind trust.

## Takeaways

- **AI is most effective on well-defined, parallelizable, verifiable tasks.** Auditing README files against TypeScript source checks all three boxes. Open-ended design work? Less so.
- **Structure makes it work.** The spec → plan → execute → review workflow is what prevented 80 inconsistent outputs. Skills provided the discipline; the AI provided the throughput.
- **AI catches what humans skip.** Not because it's smarter, but because it doesn't get bored on component #47. The value is attention to detail at scale.
- **Verification is non-negotiable.** The code review found 9 issues the agents introduced. Shipping without that pass would have replaced old inaccuracies with new ones.
- **Know which problems to point it at.** I chose this problem because it was suited for AI. Not every task is. The judgment of what to automate matters more than the tool.
- **This work would never have happened manually.** AI made it practical to tackle something that was always important but never urgent enough to prioritize.

## What's Next

This is the first post in a series I'm calling **Pair Programming with AI** — practical AI usage from a staff engineer's day-to-day work. Each post covers a real task, the approach, what worked, and what didn't.

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview), I used the [Superpowers](https://github.com/anthropics/claude-code-plugins/tree/main/superpowers) skill pack — specifically `brainstorming`, `writing-plans`, `subagent-driven-development`, and `code-review`. Try them on your own codebase — start with a well-defined, tedious task that nobody wants to do manually.

My main lesson: do not start with the flashiest task. Start with the boring one that has a clear source of truth.