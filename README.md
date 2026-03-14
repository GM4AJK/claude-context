# claude-context

A structured set of context documents used to kickstart Claude Code sessions. Rather than re-explaining the same project background, lab equipment, and workflows every time, paste the relevant files into Claude at the start of a session and get straight to work.

> **Reminder:** When starting work on a new repo, update the fine-grained GitHub personal access token to include that repo before asking Claude to create issues, PRs, or push branches. Settings → Developer settings → Personal access tokens → Fine-grained tokens.

---

## Structure

```
claude-context/
├── workflows/          # How to work: git, GitHub, requirements
├── projects/           # What to work on: per-project background docs
└── lab/                # What's on the bench: instrument control references
```

---

## workflows/

Standard processes for git, GitHub, and turning ideas into tracked requirements.

| File | Purpose |
|---|---|
| [`WORKFLOW.md`](workflows/WORKFLOW.md) | End-to-end process: requirements doc → issue → branch → implement → PR → merge |
| [`git/context-snippets.md`](workflows/git/context-snippets.md) | Ready-to-paste git commands to orient Claude at session start, before a feature, while debugging, or before a PR |
| [`requirements/fleshing-out.md`](workflows/requirements/fleshing-out.md) | How to use Claude to turn a rough idea into a well-defined GitHub issue — four stages with prompt templates |

**When to paste:** At the start of any session where you're doing git work or starting a new requirement.

---

## projects/

One file per project. Each file gives Claude the full background it needs before touching that project's code: architecture, hardware, firmware phases, current status.

| File | Project |
|---|---|
| [`TwinTorqueESC.md`](projects/TwinTorqueESC.md) | Dual-motor ESC for drones — STM32G431KBT6, register-level C, no HAL. Covers hardware stack (DRV8300, BSC014N04LS FETs), firmware phases (trapezoidal → BEMF → DSHOT → FOC), and current status. |
| [`CLAUDEDEMO.md`](projects/CLAUDEDEMO.md) | IIR filter demo on STM32G431 with automated Bode plot verification — Claude controls the FY6800 signal generator and SDS824X oscilloscope via Python to run hardware-in-the-loop tests. |

**When to paste:** Before asking Claude to work on a specific project. Only paste the relevant project file — not all of them.

---

## lab/

Control references for networked bench instruments. Paste these when Claude needs to write or debug Python scripts that talk to lab hardware.

| File | Instrument |
|---|---|
| [`SDS824X.md`](lab/SDS824X.md) | Siglent SDS824X HD oscilloscope — SCPI over TCP (`192.168.0.87:5025`). Covers time-domain measurements, raw waveform download, FFT configuration and peak search, THD calculation. |
| [`FY6800.md`](lab/FY6800.md) | Feeltech FY6800 signal generator — ASCII serial protocol over USB/CH340. Covers USB setup (usbipd-win + WSL2), command reference, frequency encoding, and a working Python control class. |

**When to paste:** When writing or debugging Python automation scripts that control instruments. The SDS824X and FY6800 docs are often needed together.

---

## How to use this repo

**Typical session flow:**

1. Open a new Claude Code session in your project repo.
2. Paste the relevant context files from this repo into the first message.
3. Describe what you want to do — Claude already has the background.

**Which files to paste:**

| Task | Paste |
|---|---|
| Starting any git/GitHub work | `workflows/WORKFLOW.md` |
| Turning an idea into an issue | `workflows/requirements/fleshing-out.md` |
| Orienting Claude in the repo | `workflows/git/context-snippets.md` output |
| Working on TwinTorqueESC | `projects/TwinTorqueESC.md` |
| Working on ClaudeDemo | `projects/CLAUDEDEMO.md` |
| Writing instrument control Python | `lab/SDS824X.md` and/or `lab/FY6800.md` |

**Keep project files current.** When a project's status changes (parts arrive, a phase completes, hardware decisions are made), update the relevant file here so the next session starts with accurate context.
