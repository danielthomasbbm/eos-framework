# EOS -- Enlightened Operating System v21.1.0

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A prompt engineering framework that shapes Claude's behavior through context-specificity: generation starts from a concrete user model instead of an implied average user. v21 added compaction-surviving state persistence via hooks and a forked slim kernel. v21.1 syncs the published kernel with the maintained live version and corrects two claims that failed the framework's own truth gate (see [Changelog](CHANGELOG.md)).

---

## What is EOS?

Large language models generate from training priors by default. Ask a generic question, get the answer for an implied average user. Adding more rules does not fix this -- rules constrain the output, but the generation still starts from generic context.

EOS takes a different approach: **context-staging**. Instead of adding rules, EOS front-loads specific user context -- domain expertise, methodology, vocabulary, project state -- so generation starts from the user's actual operating environment.

The mechanism is **specificity, not position**. A model pattern-completes from the most specific relevant context available to it. "User is experienced" displaces nothing; "user optimizes CAC on paid social against a locked $40 target" displaces the generic answer. Earlier versions of this README claimed the effect came from attention-window ordering (user context placed first dominates downstream generation). That claim was wrong as stated: causal attention means later tokens see the whole prefix, no position grants dominance, and the kernel itself sits behind the platform's own system prompt and tool definitions anyway. The USER MODEL-first ordering inside the kernel survives as a reading and maintenance convention, not a mechanism. The honest, testable claim: **specific user context in the prompt measurably reduces generic output; vague context does not.** Falsification: run the same task with a populated vs. empty USER MODEL and compare against the noun-swap test (Rule 10).

The result: responses generated against a concrete user model instead of the model's parametric defaults.

### How it works

EOS has two layers:

1. **Kernel** (`CLAUDE.md`) -- loaded as system instructions. Contains the two axioms, USER MODEL, identity declarations, compressed rule summaries, and workflow orchestration. Ordering convention: USER MODEL before Identity before Architecture before Rules. v21.1 slim core: ~270 lines / ~3,400 tokens (down from 545 lines / ~10,400 tokens in v20).

2. **Skills** -- 22 modular files that activate on specific triggers (user keywords, state transitions, metric thresholds). Each skill has its own lifecycle and references kernel rules without duplicating them. v21 adds 4 reference skills that carry full rule text, lens/sim-depth tables, runtime params, and subagent boundaries extracted from the kernel.

3. **Hooks** -- Shell scripts that fire on Claude Code lifecycle events (PreCompact, SessionStart, SessionEnd). v21 adds state persistence hooks that survive context compaction.

### What it looks like in practice

Every EOS response begins with a mandatory runtime header:

```
[lens:4] [sim-d:3] [CCI-G:65%] [sim:M] [pos:held|basis:constraint-graph] [tds:on] [ltm:2]
```

The header is a **self-check protocol, not telemetry**. The values are the model's own estimates -- there is no instrument measuring them, and a model-generated percentage is not a measurement. (Earlier versions of this README called the fields "live diagnostics." That framing failed the framework's own truth gate and is retired.) What the header actually does is force per-response attention to the things that degrade silently in long sessions: goal convergence, open assumptions, position integrity, drift. Read the values as trends -- is CCI-G rising or falling, did position move and on what basis -- not as absolute readings.

| Field | Meaning |
|-------|---------|
| `lens:4` | Context Lens setting -- how much training prior enters generation |
| `sim-d:3` | Simulation Depth -- trajectory enumeration depth |
| `CCI-G:65%` | Complete Context Index (Goal Progress) -- estimated progress toward goal convergence |
| `sim:M` | Simulation confidence -- HIGH / MEDIUM / LOW |
| `pos:held\|basis:...` | Position integrity -- whether the model held or moved its position, and why |
| `tds:on` | Tangent Drift Score -- active drift monitoring |
| `ltm:2` | Exchanges since last persistence write |

---

## Control Axes

EOS exposes two independent control axes that the user adjusts in conversation.

### Context Lens (1--5)

Controls the balance between training priors and user context in generation.

| Lens | Name | Behavior |
|------|------|----------|
| 1 | Raw Prior | Pure training distribution output. No displacement. Diagnostic mode. |
| 2 | Prior-Visible | Conventional output generated first, then user-context alternative alongside it. |
| 3 | Balanced | Both conventional and user-context paths fully developed and compared. |
| 4 | User-Led (default) | User context dominant. Convention named in one line, then bypassed. |
| 5 | Full Displacement | Maximum user context saturation. Convention gets zero tokens. |

### Simulation Depth (1--7)

Controls how many trajectories are explored and how hard each is tested.

| Depth | Name | Behavior |
|-------|------|----------|
| 1 | Surface | Single trajectory, confidence tag only. |
| 2 | Scan | Two trajectories, one failure mode each. |
| 3 | Standard (default) | All viable trajectories. One failure mode + one constraint test per path. |
| 4 | Deep | All trajectories. Two or more failure modes each. Assumptions stress-tested. |
| 5 | Adversarial | Generate strongest counterargument to recommendation. Survives or dies. |
| 6 | Constraint Sweep | For each locked constraint, reason through what relaxing it changes. (Renamed from "Monte Carlo" in v21.1 -- the model enumerates relaxations in prose, it does not run stochastic simulation. Noun-swap test, applied to ourselves.) |
| 7 | Exhaustive | All of the above. Every assumption falsification-tested. |

---

## Rules

EOS has 10 operational rules. Each handles a specific mechanical concern.

| # | Rule | Purpose |
|---|------|---------|
| 1 | Goal Lock | Goal is the only fixed point. Verified on frame shifts. Nothing starts without a clear goal. |
| 2 | Generation Frame | Generation starts from USER MODEL. Priors are reference data, not the generation seed. |
| 3 | CCI (Complete Context Index) | Living health indicator tracking goal progress and framework readiness. |
| 4 | Contradiction | Contradictions between user statements or system rules are flagged immediately. Position held until the argument changes, not the pressure. |
| 5 | Regression Lock | Resolved variables stay locked. No re-opening without new evidence. |
| 6 | Autonomy Tiers | Three tiers controlling what the system does autonomously vs. what requires user confirmation. |
| 7 | User Authority | User instructions override defaults. Hard limits are not overridable. Conflict resolution protocol. |
| 8 | Operational Empathy | Work on the problem with the user, not observe them working on it. Context-level probing over surface-level questions. |
| 9 | Context Limit Monitor | Tracks context window usage. 70% triggers state dump. 90% closes threads. |
| 10 | Output Integrity | Noun-swap test: if the output works for any user with different nouns, it is prior-derived. Regenerate from USER MODEL. |

---

## Skills

22 skill modules organized in 7 categories. Each activates on specific triggers and operates within kernel rule boundaries.

### Lifecycle
- `eos-cold-start` -- New project creation and initialization
- `eos-goal-framing` -- Goal extraction, feasibility thesis, dimension prioritization
- `eos-project-mgmt` -- Project state tracking, limiter analysis, C5 outcome tracking (prediction capture, matching, accuracy analysis)

### Build
- `eos-builder` -- Artifact-first output mode with condensed simulation

### Memory
- `eos-memory-mgmt` -- Persistence tier detection, Notion writeback policy, M1.5 skill compatibility breach protocol
- `eos-memex` -- Context compression with indexed archive and retrieval
- `eos-recall-router` -- Routes recall queries to the correct persistence tier with cross-layer dedup

### Reasoning
- `eos-contradiction` -- Dialectic contradiction handling, re-entry after rejection, C7 pattern mining (extracts hidden constraints from rejection history)
- `eos-constraint-graph` -- Constraint classification, dependency mapping, cascade unlocking, G2.6 minimization query (minimum constraint relaxation set)
- `eos-dimension-ambiguity` -- Handles ambiguous responses during dimension probing
- `eos-rules-reference` -- Full verbose Rules 1-10 text (extracted from kernel in v21 for on-demand loading)
- `eos-lens-simdepth` -- Context Lens and Simulation Depth reference tables

### Quality
- `eos-metacognition` -- F0 early warning (passive 7-signal degradation monitor), F1-F2 diagnostics, F3 rule patching with anti-churn check, F4 cross-session lessons via `tasks/lessons.md`
- `eos-fact-check` -- Claim verification against available evidence
- `eos-voice-extract` -- Session voice extraction with V2 cross-layer deduplication (checks auto-memory, Notion, Pieces before writing)
- `tangent-drift-score` -- Goal drift detection with context-specific sarcastic redirects

### Output
- `eos-report` -- Structured deliverable generation with multi-source cross-referencing

### System
- `eos-collaboration` -- Multi-user session coordination and attribution
- `eos-multi-agent` -- 7-phase agent lifecycle: pre-flight, recon, decomposition, deployment, Phase 3.5 cross-agent validation, consolidation, Phase 4.5 reconciliation audit
- `eos-kernel-updater` -- Kernel self-modification with Step 1.5 patch churn detection (3+ patches on same rule triggers structural review)
- `eos-runtime-params` -- Full runtime parameters reference table
- `eos-autonomy-boundaries` -- Subagent execution boundaries and tool budget enforcement

---

## Operating Modes

### Builder Mode

Activated by "let's build," "start coding," or "build mode on." Output shifts to artifacts -- code, documents, deliverables. Clarifying questions suppressed except genuine blockers. Simulation condensed to one-line confidence notes. Runtime header still required.

### Situational Awareness

EOS operates as a persistent partner across all active work. On any task: checks the active project landscape, maps tasks to the right project, surfaces cross-project conflicts, triages random input to the right branch.

### Cross-Session Learning

`tasks/lessons.md` captures self-correcting rules from every user correction. Loaded at session start as behavioral constraints. Written immediately on correction (not batched). 3+ occurrences across distinct sessions escalates to structural review.

---

## Quick Start

### 1. Copy the kernel

```bash
cp kernel/CLAUDE.md ~/.claude/CLAUDE.md
```

### 2. Copy skills

```bash
cp -r skills/ ~/.claude/skills/
```

### 3. Validate

```bash
bash tools/validate-skills.sh
```

### 4. Start a conversation

Open Claude Code or claude.ai. EOS activates automatically. You will see a runtime header on every response and the system will ask for a goal before substantive work begins.

See [docs/quick-start.md](docs/quick-start.md) for a detailed walkthrough. See [docs/installation.md](docs/installation.md) for platform-specific setup.

---

## Structure

```
eos-framework/
  kernel/
    CLAUDE.md              # Slim kernel (~270 lines). System instructions loaded at session start.
  skills/
    lifecycle/             # Cold start, goal framing, project management
    build/                 # Artifact construction
    memory/                # Persistence, recall, knowledge graph
    reasoning/             # Contradiction, constraints, ambiguity, rules reference, lens/sim-depth
    quality/               # Metacognition, fact-check, voice, drift
    output/                # Report generation
    system/                # Collaboration, multi-agent, kernel updates, runtime params, boundaries
  docs/
    architecture.md        # Deep-dive on context-staging philosophy
    installation.md        # Platform-specific setup instructions
    quick-start.md         # 5-minute setup guide
    skill-authoring.md     # How to write new skills
  hooks/
    eos-precompact.sh      # State backup before context compaction
    eos-session-start.sh   # State injection on session start / post-compaction
    eos-session-end.sh     # Final state backup on session close
    credential-guard.sh    # Blocks edits to .env / credential files
    file-backup.sh         # Auto-snapshots before file mutations
    search-year-fix.sh     # Appends current year to web searches
  tools/
    validate-skills.sh     # Validates skill integrity against kernel
  examples/
    user-config/           # Example USER MODEL configurations
    memory/                # Example persistence tier setups
```

---

## Documentation

- [Architecture](docs/architecture.md) -- Context-staging philosophy, token ordering, compression prohibition
- [Installation](docs/installation.md) -- Platform-specific setup for Claude Code and claude.ai
- [Quick Start](docs/quick-start.md) -- 5-minute setup and first conversation
- [Skill Authoring](docs/skill-authoring.md) -- How to write new EOS skills
- [Hooks](hooks/README.md) -- Safety and quality hooks for Claude Code

---

## Hooks

Six Claude Code hooks ship with EOS -- three for state persistence (v21) and three for safety/quality:

| Hook | Event | Purpose |
|------|-------|---------|
| `eos-precompact.sh` | PreCompact | Backs up EOS state file before context compaction, injects recovery message |
| `eos-session-start.sh` | SessionStart | Injects state file content on session start and post-compaction reload |
| `eos-session-end.sh` | SessionEnd | Final state backup on session close |
| `credential-guard.sh` | PreToolUse | Blocks Write/Edit on `.env`, credential files, private keys |
| `file-backup.sh` | PreToolUse | Creates timestamped backup before any file mutation |
| `search-year-fix.sh` | PreToolUse | Appends current year to web searches for fresh results |

See [hooks/README.md](hooks/README.md) for installation instructions.

---

## Known Issues

- **Pieces LTM reference drift.** Pieces MCP supplementary writes can drift from Notion state over long sessions. Notion is authoritative when they conflict. Pieces is a convenience layer, not a source of truth.
- **Context window pressure.** The slim kernel is approximately 3,400 tokens (~270 lines). v21 reduced this from ~10,400 tokens by moving verbose content to on-demand skill files. The Context Limit Monitor (Rule 9) manages context pressure, but users in extended sessions should monitor the `ltm` counter in the runtime header.
- **Platform overlap with native memory.** Claude Code now ships native auto-memory (`MEMORY.md` plus memory files) and compaction-surviving conversation summaries. These overlap with the v21 state persistence hooks. The hooks remain the durable, schema-controlled copy -- native summaries are lossy and not user-controlled -- but expect this machinery to shrink in future versions as the platform absorbs it. The durable parts of EOS are the behavioral rules (falsification discipline, goal lock, noun-swap test), not the state plumbing.
- **Skill module loading.** On claude.ai (non-Code), skill loading depends on the Projects feature placing files in `/mnt/skills/user/`. Verify paths if skills are not triggering.
- **State file prompt cache impact.** The SessionStart hook injects state file content as a systemMessage. Whether this breaks Claude Code's prompt cache prefix stability depends on where hook systemMessages are inserted in the API request. Empirical testing needed -- if cache breaks are detected, state injection can move to a file read on the first model turn instead.
- **Compaction survival is best-effort.** The state file captures what the model wrote during the conversation. If the model fails to write state on a change event, that state is lost. The PreCompact hook warns on stale state (>5min).

---

## License

Apache License 2.0. See [LICENSE](LICENSE) for details.
