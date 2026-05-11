---
name: architectural-refactor-reasoning
description: >
  Use this skill whenever the user asks Claude to plan, propose, or execute a significant
  architectural refactor of a codebase — especially when it involves restructuring component
  hierarchies, decoupling data from UI, migrating state management, splitting monolith
  files, reorganizing a module system, or extracting shared logic into reusable units.
  Trigger on phrases like "refactor this", "make this scalable", "this file is too big",
  "decouple this", "clean up the architecture", "make this easier to maintain", "I want
  to be able to reuse this", or any request where the user has a working feature that
  needs a structural overhaul rather than a patch. Also trigger when the user pastes
  complex coupled code and asks how to make it more flexible or testable.
  ALWAYS use this skill before proposing any multi-phase refactor — never wing it on
  architecture decisions.
---

# Architectural Refactor Reasoning

This skill encodes a forensically extracted thought process for planning and executing
complex codebase refactors. It is not a checklist — it is a *way of thinking*. Follow
the reasoning pattern below as an internal mental model before producing any output.

---

## ⚠️ Ask This Before Anything Else: Should We Even Refactor?

Before proposing a single structural change, honestly assess whether the refactor
is the right call *right now*. Refactoring has real costs: time, risk of regression,
cognitive load, and the danger of breaking something that was working.

**Do not proceed with a refactor if any of the following are true — flag them first:**

- **The codebase is in active production with no test coverage.** A refactor here
  is live surgery with no safety net. Name this risk explicitly.
- **The pain is localized.** If only one file is messy and nothing else touches it,
  a targeted patch is safer than a structural overhaul.
- **The deadline is closer than the refactor's risk window.** Shipping beats clean.
- **The user hasn't fully understood the current behavior.** Refactoring code you
  don't fully understand is how silent bugs are born.
- **The coupling is load-bearing.** Some tight coupling exists for performance or
  intentional reasons. Decoupling it may introduce latency or complexity elsewhere.
- **The scope is unclear.** If you can't draw a boundary around what changes and
  what doesn't, the refactor will expand until it breaks something unrelated.

**How to handle it:**

Present the risk clearly and honestly. Don't bury it. Say:

> "Before we proceed: I want to flag that this refactor carries [specific risk].
> The current code is working. The danger of [X] is real. Here are your options:
> (A) refactor with controlled phases, accepting the risk; (B) targeted patch for
> the immediate problem only; (C) leave it and document the debt.
> What's your call?"

If the user understands the risk and chooses to proceed — proceed fully and without
hedging. Your job was to inform, not to block.

---

## The Core Reasoning Pattern (in order)

### 1. Understand the True Goal First

Before touching structure, answer: **what does the user actually want to achieve?**

- Is it reusability? Testability? Scalability? Readability? Performance? Team velocity?
- The stated request is often surface-level. Dig one layer deeper.

Examples of surface vs. real goal:
- "Split this file" → real goal: *"I need two different teams to own different parts"*
- "Extract this logic" → real goal: *"I want to test this without spinning up a database"*
- "Make this more modular" → real goal: *"I need to swap implementations per environment"*

Name the real goal in one sentence before proposing anything.

### 2. Diagnose the Current Architecture's Pain

Identify **why the current structure causes the problem**. Be specific:

- **Tight coupling** — module A directly instantiates module B; can't test A without B
- **Monolith file** — one class/file doing authentication, business logic, and formatting
- **Nested or entangled state** — shared mutable state makes execution order unpredictable
- **Mixed concerns** — data fetching, transformation, and rendering in the same function
- **Implicit dependencies** — behavior changes based on global variables or singletons
- **Duplicated logic** — same validation or formatting logic copy-pasted in 4 places

Don't just say "it's messy." Name the specific structural problem.

### 3. Identify the Key Architectural Insight

There is always one or two **key insights** that unlock the refactor. Find them before
proposing the solution. This is the most important step — a coherent mental model
before generating structure.

Examples across different domains:

- *Form/UI:* "Steps are just arrays of field IDs — moving a field means editing config, not components."
- *Service layer:* "The business logic doesn't know it's talking to a database — inject a repository interface instead."
- *State management:* "Derived values don't need to live in state — compute them from a single source of truth."
- *API layer:* "Request validation, transformation, and response shaping are three separate concerns living in one function."
- *Monolith split:* "These two domains share a database table but nothing else — the table is the only thing blocking separation."

Do not propose a solution until you've articulated the key insight in one sentence.

### 4. Identify the Cross-Cutting Concerns Early

Before designing the solution, enumerate what **resists clean decomposition**. These
are the **architecture traps** — things that look like they can be separated but have
hidden dependencies:

- Shared mutable state that multiple modules read and write
- Side effects triggered by seemingly unrelated operations
- Logic that must run in a specific order across module boundaries
- Error handling that spans multiple layers
- Auth, logging, or observability woven through business logic
- Configuration or environment values accessed deep in the call stack

Design the solution to handle these explicitly, not as afterthoughts.

### 5. Propose a Concrete Structure with Intention-Revealing Names

Name the **actual files, classes, and functions** you'll create. A name should answer:
*what does this do?* — not where it lives in the file tree.

**Naming discipline — non-negotiable:**
- Every new module, class, function, and file must have a name that reveals its intent
- A reader should understand what it does before opening it
- If you can't name it without using "Helper", "Utils", "Manager", or "Handler" — the
  abstraction boundary isn't clear yet. Rethink the boundary, then name it.

| Generic (bad) | Intention-revealing (good) |
|---------------|---------------------------|
| `helper.js` | `formatCurrencyForDisplay.js` |
| `UserManager` | `UserSessionAuthenticator` |
| `DataProcessor` | `normalizeApiResponseToViewModel` |
| `utils/index.js` | `validators/requireNonEmptyString.js` |
| `BaseComponent` | `PaginatedDataTable` |

Name every file and module in your proposal before writing code.

### 6. Handle the Middle Ground Explicitly

Acknowledge when a *full* refactor conflicts with *right now* constraints. Don't pretend
the tension doesn't exist. Find the minimum viable architecture that solves the real goal:

- What must change now to unblock the goal?
- What can stay coupled for now without causing harm?
- What's the simplest pattern that doesn't introduce new brittleness?

The cleanest architecture isn't always the right one for today. A thin abstraction
layer or a single extracted module may be enough — and safer than a full restructure.

### 7. Propose Incremental Phases with Verification Gates

Never propose a big-bang refactor. Always phase it. **Complete one phase fully before
starting the next** — no half-finished states, no parallel tracks.

Each phase must define:
- **Goal** — one sentence describing what this phase achieves
- **Scope** — what changes, what explicitly does not
- **Acceptance criteria** — specific, verifiable conditions (not just "build passes")
- **Verification** — what the user should check before approving the next phase

**Phase template (adapt to the actual codebase):**

```
Phase 0: Inventory + contracts + skeleton (zero behavior change)
         Goal: Understand the full blast radius before touching anything
         Scope: Read-only analysis + new empty files/interfaces
         ✓ Acceptance: No existing file modified. New structure documented.
                       Build passes. Team has reviewed the plan.

Phase 1: Extract [the first logical unit]
         Goal: [specific thing being separated]
         Scope: [exactly which files change]
         ✓ Acceptance: [concrete behavior that must still work]
                       [specific thing that must not break]
                       Build + lint pass clean.

Phase 2: [Next logical unit]
         ...

Phase N: Remove old code + verify
         ✓ Acceptance: No dead imports. No unused exports.
                       All original behavior preserved. Build + lint clean.
```

User approves each phase before the next begins. This is non-negotiable.

### 8. Protect Existing Callers

Before executing any phase that changes a module's interface or data shape:

- Identify every caller of the code being changed
- Determine if the interface change is breaking or additive
- If breaking: design a **compatibility adapter or migration path** so callers
  don't all need to change at once
- Name the adapter with intention: `adaptLegacyUserShapeToProfileModel()`, not `adapter()`
- Declare explicitly: "These callers will not need to change" or "These N files need updating"

Never let a refactor silently break a caller.

### 9. Apply the Rule of Deferred Genericity

Don't over-engineer the solution to handle cases that don't exist yet.

- Two known variants? Use a config object or a parameter before building a plugin system.
- One edge case? Handle it as a conditional, not a new abstraction layer.
- Anticipated future requirements? Build for what exists now, structure so it's extensible.

*Generic = more indirection, harder to debug, harder to change.*
*Specific + modular = less indirection, easier to test, easier to change.*

The right level of abstraction is the one that solves today's problem without
making tomorrow's problem harder.

### 10. State What You're Not Doing (and Why)

A good refactor proposal names its **excluded options** explicitly. This prevents
scope creep and builds trust.

Examples:
- "Not migrating the database layer — only the service interface changes."
- "Not extracting the auth logic yet — it's out of scope for this goal."
- "Not making this generic for all entity types — only the two that exist now."
- "Not changing the public API contract — all callers stay untouched."

---

## Output Format for Every Refactor Proposal

**Always propose the full plan before writing any code. Never start implementing
while the plan is unapproved.**

```
## Risk Assessment
[Is this refactor safe to do now? Which of the six risk flags apply, if any?]

## Real Goal
[One sentence — the actual thing the user needs, not just the stated request]

## Current Problem
[Named structural issue — specific, not "it's messy"]

## Key Architectural Insight
[The unlock — one or two sentences that make the solution obvious]

## Cross-Cutting Concerns
[What resists clean separation and how we'll handle each one]

## Proposed Structure
[Every new file/module/class with intention-revealing names and estimated line counts]

## Phases
[Each phase: goal + scope + acceptance criteria + verification step]

## What We're Not Doing
[Explicit exclusions with reasons]

## Questions Before Executing
[Ambiguities that need user sign-off before a single line is written]
```

Do NOT skip Risk Assessment or Questions.
Do NOT write code until the plan is approved.

---

## Anti-Patterns to Avoid

- **Skipping the risk check** — always assess whether refactoring is the right call first
- **Big-bang refactor** — one massive PR that changes everything at once
- **Starting Phase 2 before Phase 1 is verified** — finish what you started
- **Vague proposals** — "we'll extract the logic" without naming what, where, and how
- **Generic names** — `Helper`, `Utils`, `Manager`, `Handler`, `Component` — name the intent
- **Silent interface changes** — modifying a module's contract without updating its callers
- **Over-engineering** — building plugin architectures for two known variants
- **Ignoring load-bearing coupling** — assuming all coupling is bad and should be removed
- **Implementing before planning** — writing code before the proposal is approved

---

## When the User Says "Do It Now"

Acknowledge the tension. Then say:

> "I can start immediately — but I want 60 seconds of your attention on the phase plan
> before I touch any files. It's the difference between a clean refactor and a week of
> debugging. Phase 0 is zero-risk. Here's the plan:"

Show the full plan. Get approval. Execute Phase 0 only.

---

## Key Heuristics

| Situation | Default response |
|-----------|-----------------|
| Logic appears in multiple places | Extract to a single named function/module with one clear owner |
| Two modules directly depend on each other | Introduce an interface or shared abstraction between them |
| Behavior changes based on global/singleton state | Inject the dependency explicitly instead |
| One file is doing too many things | Identify the distinct responsibilities; one module per responsibility |
| Caller needs to be isolated for testing | Extract to an interface the test can stub |
| Two variants of the same thing exist | Config/param first; separate modules only if divergence is significant |
| Old callers use a different data shape | Compatibility adapter with an intention-revealing name; callers unchanged |
| Tempted to use a generic name | Stop — rethink the abstraction boundary until the name becomes obvious |
| Refactor feels risky | Run the risk assessment; present options; let the user decide |
| User says "do it now" | Plan first, 60 seconds, then Phase 0 |

---

## File Size Reference

These are guidelines, not rules. The goal is a file size where the entire module
fits in working memory — you can read it start to finish and understand what it does.

| Module type | Comfortable range | Split signal |
|-------------|------------------|--------------|
| Pure utility / single function | 10–50 lines | Rarely needs splitting |
| Single-concern module | 50–150 lines | At 200+ if concerns diverge |
| Orchestrator / coordinator | 150–300 lines | At 400+ extract sub-concerns |
| Complex stateful component | 200–400 lines | At 500+ always split |
| Config / registry file | 30–100 lines | At 150+ split by domain |

If a file will exceed 500 lines, identify the split point before starting.
