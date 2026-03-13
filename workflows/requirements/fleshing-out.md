# Fleshing Out a Requirement with Claude

How to take a rough idea — a problem you noticed, a feature you want, a measurement you need to make — and turn it into a well-defined GitHub issue with a linked requirements doc.

---

## Overview

| Stage | Goal | Output |
|---|---|---|
| 1. Explore | Understand the idea | Shared mental model |
| 2. Define | Write a formal requirement | Requirements doc |
| 3. Issue | Track the work | GitHub issue |
| 4. Branch | Start implementation | Feature branch |

---

## Stage 1: Exploring the Idea with Claude

Start with whatever you have — even a rough sentence. The goal is to have Claude ask you good questions and help you understand the scope before you commit to anything.

**Prompt template:**

```
I have a rough idea I want to explore. Don't write any code or create any documents yet —
just ask me questions to help me understand what I actually need.

The idea: <describe it in a sentence or two>

Things I already know:
- <any constraints you have>
- <any prior attempts>

Ask me up to 5 clarifying questions before we proceed.
```

**What to expect:** Claude will probe the edges of the idea — what "done" looks like, what can go wrong, whether similar solutions already exist, what the acceptance criteria might be.

**When to move on:** When you can answer the question "how will I know this works?" with a specific, testable answer.

---

## Stage 2: Defining the Requirement Formally

Once the idea is clear, ask Claude to write a structured requirements document. This becomes the specification — write it before any code.

**File location convention:**
```
Docs/requirements/N-slug/README.md
```

**Prompt template:**

```
Based on our discussion, write a requirements document for this feature.
Use this exact structure:

## Goal
One sentence describing what this achieves.

## Background
Why this is needed. What problem it solves.

## Scope
What is in scope. What is explicitly out of scope.

## Specification
Precise description of the behaviour, interface, or measurement required.
Include units, ranges, timing constraints, or protocol details where relevant.

## Test Setup
What hardware, software, or environment is needed to verify this.

## Measurement Method
Step-by-step: how to run the test and capture results.

## Acceptance Criteria
Numbered list of pass/fail conditions. Each one must be objectively verifiable.

## Open Questions
Anything still unresolved.

Save this to: Docs/requirements/<N>-<slug>/README.md
```

**Review before moving on:** Read the acceptance criteria carefully. If any of them are vague ("works correctly", "performs well"), push back and ask Claude to make them precise.

---

## Stage 3: Turning It into a GitHub Issue

Once the requirements doc is committed, create a GitHub issue that links to it. The issue is the unit of tracking — the doc is the source of truth.

**Prompt template:**

```
Write a GitHub issue for the requirement we just documented.

Use this format:

**Title:** <verb phrase — e.g. "Add PWM frequency control via UART command">

**Body:**

## Summary
One paragraph describing the feature and why it's needed.

## Specification
Link to the requirements doc: `Docs/requirements/<N>-<slug>/README.md`

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] <criterion 3>

## Notes
Any implementation hints, constraints, or related issues.

Then run:
gh issue create --title "<title>" --body "<body>"

Note the issue number — I'll use it for the branch name and PR.
```

**Tip:** Use checkboxes in the acceptance criteria (`- [ ]`). GitHub renders these as a progress tracker on the issue card.

---

## Stage 4: Creating a Branch from the Issue Number

Once you have the issue number, create a branch that references it. This keeps the git history linked to the issue.

```bash
# Replace N with the issue number and slug with a short description
git checkout -b feature/N-slug

# Example:
git checkout -b feature/42-uart-pwm-control
```

**Convention used in this repo:**
```
feature/<issue-number>-<short-slug>
```

When you open the PR later, reference the issue with `Closes #N` in the body and GitHub will auto-close the issue on merge.

---

## Tips

**Don't skip Stage 1.**
Jumping straight to writing a requirements doc for an underspecified idea produces a polished document about the wrong thing. The exploration stage is where you discover the real requirement.

**Write the doc before the code.**
The requirements doc is a forcing function. If you can't write a clear acceptance criterion, you don't understand the requirement well enough to implement it.

**One requirement per issue.**
If the requirements doc covers two independent things, split it. Small, focused issues are easier to review, easier to merge, and easier to close.

**Commit the doc first, then implement.**
This gives you a clean commit history: `Add Req N requirements doc` → `Implement Req N`. Anyone reading the log can see what was specified before they see what was built.

**Use the issue number in every related artifact.**
Branch name, PR title, commit messages, and test file names should all reference the issue number. This makes it trivial to trace any file back to its requirement.

**Let Claude draft, you decide.**
Claude is good at producing structured text quickly. But you own the acceptance criteria — read them carefully and push back if anything is hand-wavy. "Signal is stable" is not an acceptance criterion. "Signal settles to within ±1% of target within 10 ms" is.
