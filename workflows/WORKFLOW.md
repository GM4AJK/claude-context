# Git Workflows

Standard workflows for my projects including interacting with Git and GitHub

---

## For each feature

### 1. Draft the design spec — discuss before coding

```
specs/<slug>.md
```

Written through discussion with Claude, not handed over ready-made: talk through the goal,
approach/design, and acceptance criteria, iterating on the doc until it's right. This is the
specification, and it comes before any code.

**Do not start implementing until the spec is agreed.** If Claude is unsure whether a spec is
settled, it should ask rather than assume and start coding.

### 2. Commit and push the spec

```bash
git add specs/<slug>.md
git commit -m "Add design spec: <slug>"
git push
```

### 3. Create a feature branch

```bash
git checkout -b feature/<slug>
```

### 4. Implement firmware changes

- Edit `Core/Src/main.c` and/or `Core/Src/stm32g4xx_it.c` inside
  `USER CODE BEGIN` / `USER CODE END` blocks
- Peripheral init in static functions called from `USER CODE BEGIN 2`
- Bare-metal register writes only — no HAL peripheral APIs

### 5. Write a verification script, if the feature needs one

No fixed location — place it wherever makes sense for that feature (alongside the spec, in an
existing test directory, etc.). Not every feature needs one.

### 6. Commit and push the implementation

```bash
git add <changed files> <verification script, if any>
git commit -m "Implement <slug>: <short description>"
git push -u origin feature/<slug>
```

### 7. Open a pull request

```bash
gh pr create --title "<title>" --body "Implements specs/<slug>.md"
```

### 8. User builds and tests on hardware

The user builds in STM32CubeIDE and runs any verification script. Claude does not build
or flash — the user does this. State the run command (if any) in the PR description.

### 9. Merge on pass

Once verified:

```bash
gh pr merge <PR number> --squash --delete-branch
git checkout main
git pull
```

---

## Commit conventions

- No GPG signing required — commit directly
- Use a heredoc for multi-line messages to preserve formatting

## Git remote

The `gh` CLI is authenticated and can create PRs directly.
