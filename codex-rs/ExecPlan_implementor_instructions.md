# ExecPlan Implementation Instructions

These instructions guide coding agents who are executing work defined in an existing ExecPlan. Follow them alongside the ExecPlan you are implementing and the repository’s current state.

## 1. Read and Internalize Before Executing

- Read the entire ExecPlan end to end before making any edits. It is self-contained — rely solely on it and the repository.
- Study the `Purpose / Big Picture`, `Context and Orientation`, and `Plan of Work` sections until you can restate the objective and intended end-to-end behavior in your own words.
- Identify every milestone and acceptance criterion so you know exactly how progress will be measured and confirmed.
- Do not request “next steps.” Proceed autonomously unless the user has explicitly limited the scope (e.g., “complete steps 1–3”).

## 2. Keep the ExecPlan Updated and Self-Contained

- Treat the ExecPlan as your single source of truth. Update it immediately whenever you complete a step, check off a `Progress` item, make a design decision, or learn something new. More frequent updates are better.
- Each revision must leave the ExecPlan fully self-contained. If someone could restart from scratch using only the updated plan, you have done it correctly.
- Never defer updates. If you learn something during a run or investigation, capture it right away — put evidence in `Surprises & Discoveries` or note it in the `Decision Log` before moving on.
- Record any new assumptions, dependencies, or prerequisites you uncover so the plan remains hermetic for a clean restart at any point.

## 3. Maintain the `Progress` Checklist

- Log every stopping point in `Progress` using the checkbox format already defined in the ExecPlan, along with a timestamp (`date "+%m-%d %H:%M"`).
- If a task is only partially complete, split it into “completed” and “remaining” bullets so the current state is always explicit.
- When resuming work, review `Progress` first to regain context. If it does not clearly indicate the next action, clarify or expand the entry before continuing.

## 4. Capture Decisions and Insights

- Log every choice that alters design, scope, or implementation details in the `Decision Log`, including its rationale, author, and timestamp.
- Record unexpected results, optimizer behavior, difficult bugs, or measurements in `Surprises & Discoveries`, supported by concise evidence (snippets or command output).
- When the user directs a change, create a `Decision Log` entry authored as "User".
- Keep both sections synchronized with your actual actions so future readers can understand how and why the plan evolved.

## 5. Execute Milestones as Written

- Follow the milestones in `Plan of Work` sequentially unless you revise the plan to document a justified change in order.
- For prototype or exploratory milestones, record the acceptance signals you observed (tests, command output) and summarize what you concluded before proceeding.
- When a milestone is complete, ensure `Progress`, `Concrete Steps`, `Validation and Acceptance`, and `Outcomes & Retrospective` all accurately reflect the new state.

## 6. Update Commands and Validation Proof

- Record all **significant** commands in `Concrete Steps`, including the working directory and any required environment details. Omit trivial commands (`rg`, `cat`, `date`, `sed`, `git show`, `ls`, etc.).
- If a command produces output that proves success, capture a concise excerpt in `Artifacts and Notes` and keep it reproducible.
- When tests, CLIs, or services change, update `Validation and Acceptance` so that a future implementor can reproduce the verification without guesswork.

## 7. Preserve Safety, Idempotence, and Recovery Paths

- Ensure the guidance in `Idempotence and Recovery` remains accurate whenever you add steps that can fail or modify state.
- Document safe rerun or rollback procedures **before** performing any operation that carries risk.
- Favor additive, reversible work until cleanup is explicitly required; record any deviations and how to recover.

## 8. Maintain Code Pointers and Dependencies

- When you touch new files, modules, or external APIs not already listed, add them to `Code Pointers` or `Interfaces and Dependencies` with precise paths and context.
- Remove outdated pointers only after replacing them with updated, authoritative locations.

## 9. Close the Loop

1. When a major milestone - or the entire ExecPlan - completes, add an entry to `Outcomes & Retrospective` summarizing results, remaining gaps, and key lessons.
2. Verify that all sections are consistent, then walk through the plan as if starting from scratch to confirm it remains coherent and complete.
3. If handoff is required, ensure the updated ExecPlan alone enables the next agent to continue seamlessly.
