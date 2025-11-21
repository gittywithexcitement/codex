# Preface

This document describes the requirements for an execution plan (hereafter "ExecPlan"), a design
document that a coding agent can follow to deliver a working feature or system change.

# How to author an ExecPlan

## Authoring

When authoring an ExecPlan, follow this file, ExecPlan_authoring.md, _to the letter_. If it is
not in your context, refresh your memory by reading the entire ExecPlan_authoring.md file. Be
thorough in reading (and re-reading) source material (e.g. code in this repository) to produce
an accurate plan. When creating a ExecPlan, start with an outline and flesh it out as you do
your research.

## Requirements

STRICT REQUIREMENTS:

- ExecPlans must be fully self-contained. Self-contained means that in its current form it
  contains all knowledge and instructions needed for a novice coder to succeed; there is no
  other context provided to the novice.
- Each ExecPlan is a living document. Implementors must revise it as progress is made, as
  discoveries occur, and as design decisions are finalized. Each revision must remain fully
  self-contained.
- ExecPlans must enable a complete novice coder to implement the feature end-to-end without
  prior knowledge of this repository.
- ExecPlans must produce demonstrably working behavior, not merely code changes to "meet a
  definition".
- ExecPlans must define every term of art in plain language (i.e. define jargon).

Purpose and intent come first. Begin by explaining, in a few sentences, why the work matters from a
user's perspective: what someone can do after this change that they could not do before, and how to
see it working. Then guide the reader through the exact steps to achieve that outcome, including
what to edit, what to run, and what they should observe.

### Hermeticity

Treat the reader as a complete novice to this repository: they have only the current code and
the ExecPlan file you create. There is no memory of prior plans and no external context.

The ExecPlan and the current state of the repository are the only knowledge that an implementor
will have. The agent executing your plan can list files, read their contents, search for files
by name, search for files containing text patterns, and run tests and binaries. It does not know
any prior context and does not have context gleaned from completing earlier milestones (thus,
knowledge gained during completion of a milestone that is relevant to a later milestone must be
recorded in the ExecPlan). Record all assumptions you rely on. Do not point to external blogs or
documents; if knowledge is required, embed it in the ExecPlan itself in your own words. If an
ExecPlan builds upon a prior ExecPlan in the repository, incorporate it by reference. Otherwise
include all relevant context from that plan.

## Formatting

Follow Markdown syntax. Include a blank line after headers, use # and ## and so on, and correct
syntax for ordered and unordered lists.

Write in plain prose in narrative sections (i.e. Purpose, Outcomes, Context, Plan of Work,
Idempotence). Prefer sentences over lists. Avoid checklists, tables, etc.. Checklists are
permitted only in the `Progress` section, where they are mandatory.

## Guidelines

Self-containment and plain language are paramount. Define jargon immediately and remind the
reader how it manifests in this repository (for example, by naming the files or commands where
it appears). Do not say "as defined previously" or "according to the architecture doc." Include
the needed explanation here, even if you repeat yourself.

Do not describe "the letter of a feature" so narrowly that the resulting code compiles but does
nothing meaningful. Do not outsource key decisions to the reader. When ambiguity exists, resolve
it in the plan itself and explain why you chose that path. Err on the side of over-explaining
user-visible effects and under-specifying incidental implementation details.

Anchor the plan with observable outcomes. State what the user can do after implementation, the
commands to run, and the outputs they should see. Acceptance should be phrased as behavior a
human can verify (e.g. "after starting the server, navigating to
[http://localhost:8080/health](http://localhost:8080/health) returns HTTP 200 with body OK")
rather than internal attributes ("added a HealthCheck struct"). If a change is internal, explain
how its impact can still be demonstrated (for example, by running tests that fail before and
pass after, and by showing a scenario that uses the new behavior).

Specify repository context explicitly. Name files with full repository-relative paths, name
functions and modules precisely, and describe where new files should be created. If touching
multiple areas, include a short orientation paragraph that explains how those parts fit together so
a novice can navigate confidently. When running commands, show the working directory and exact
command line. When outcomes depend on environment, state the assumptions and provide alternatives
when reasonable.

Be idempotent and safe. Write the steps so they can be run multiple times without causing damage
or drift. If a step can fail part way through, include how to retry or adapt. If a migration or
destructive operation is necessary, spell out backups or safe fallbacks. Prefer additive,
testable changes that can be validated as you go.

Validation is required. Include instructions to run tests, to start the system if applicable,
and to observe it doing something useful. Describe comprehensive testing for any new features or
capabilities. Include expected outputs and error messages so a novice can tell success from
failure. Where possible, show how to prove that the change is effective beyond compilation (for
example, through a unit or integration test, small end-to-end scenario, a CLI invocation, or an
HTTP request/response transcript). State the exact test commands appropriate to the project’s
toolchain and how to interpret their results.

Capture evidence. When your steps produce terminal output, short diffs, or logs, include the
evidence inside a single fenced block. Keep it concise and focused on what proves success. If
you need to include a patch, prefer file-scoped diffs or small excerpts that a reader can
recreate by following your instructions rather than pasting large blobs.

When you revise an ExecPlan, you must ensure your changes are comprehensively reflected across
all sections.

## Milestones

Milestones are narrative. If you break the work into milestones, introduce each with a brief
paragraph that describes the scope, what will exist at the end of the milestone that did not
exist before, the commands to run, and observable acceptance criteria. Keep it readable as a
story: goal, work, result, proof. Progress and milestones are distinct: milestones tell the
story, progress tracks granular work. Both must exist. Never abbreviate a milestone merely for
the sake of brevity; do not leave out details that could be crucial to a future implementation.

Each milestone must be independently verifiable and incrementally implement the overall goal of the
execution plan.

## Living plans and design decisions

- ExecPlans must contain and maintain all of the sections described in "ExecPlan Template".
- ExecPlans are living documents. As you make key design decisions, update the plan to record both
  the decision and the thinking behind it. Record all decisions in the `Decision Log` section.
- When you discover optimizer behavior, performance tradeoffs, unexpected bugs, or inverse/unapply
  semantics that shaped your approach, capture those observations in the `Surprises & Discoveries`
  section with short evidence snippets (test output is ideal).
- If you change course mid-implementation, document why in the `Decision Log` and reflect the
  implications in `Progress`. Plans are guides for the next contributor as much as checklists for
  you.
- At completion of a major task or the full plan, write an `Outcomes & Retrospective` entry
  summarizing what was achieved, what remains, and lessons learned.

## Prototyping milestones and parallel implementations

It is acceptable - and often encouraged - to include explicit prototyping milestones when they
de-risk a larger change. Examples: adding a low-level operator to a dependency to validate
feasibility, or exploring two composition orders while measuring optimizer effects. Keep
prototypes additive and testable. Clearly label the scope as “prototyping”; describe how to run
and observe results; and state the criteria for promoting or discarding the prototype.

Prefer additive code changes followed by subtractions that keep tests passing. Parallel
implementations (e.g., keeping an adapter alongside an older path during migration) are fine
when they reduce risk or enable tests to continue passing during a large migration. Describe how
to validate both paths and how to retire one safely with tests. When working with multiple new
libraries or feature areas, consider creating spikes that evaluate the feasibility of these
features _independently_ of one another, proving that the external library performs as expected
and implements the features we need in isolation.

# Recap

If you follow the guidance in this document, a single, stateless agent - or a human novice - can
read your ExecPlan from top to bottom and produce a working, observable result. These are the
requirements:

- self-contained
- self-sufficient
- implementable by a novice
- outcome-focused

# Where to write the ExecPlan

Write the ExecPlan as a Markdown file, in the `todo/` directory (create one as needed) where the
agent (you) were started. Name the plan file <change_summary>_plan.md, e.g.
`todo/database_migration_plan.md`.

# ExecPlan Template

- Headers must be copied into the template verbatim. <instructions in brackets> must be filled
  in by you.
- All other text is a directive for you to interpret and place at this location in the resulting
  ExecPlan.
- Everything below the ---- line is a template that you should start with when writing an
  ExecPlan.

----

# <Short, action-oriented description>

Copy the following verbatim **except** replace the link target with the absolute path produced
by running `realpath ExecPlan_implementor_instructions.md` (the file is a sibling of this one,
`ExecPlan_authoring.md`):

Read the ExecPlan implementation guide at
[ExecPlan_implementor_instructions.md](/absolute/path/to/ExecPlan_implementor_instructions.md)
now. Follow its instructions and keep this document updated as work proceeds.


## Purpose / Big Picture

Write a concise explanation of what the user gains after this change and how they can see it
working. State the user-visible behavior you will enable. This section will likely not change as
implementation progresses.

## Progress

Use a list with checkboxes to summarize granular steps. Every stopping point must be documented
here, even if it requires splitting a partially completed task into two (“done” vs.
“remaining”). This section must always reflect the actual current state of the work. Copy this
first step verbatim:

- [ ] Read ExecPlan_implementor_instructions.md (see link above) and follow its
  instructions carefully as you implement this ExecPlan.
- [x] (10-31 10:31) Example completed step.
- [ ] Example partially completed step (completed: Foo; remaining: Bar).
- [ ] Example incomplete step.

Add a timestamp (`date "+%m-%d %H:%M"`) when checking off a task.

## Surprises & Discoveries

Document unexpected behaviors, bugs, optimizations, or insights discovered during implementation.
Provide concise evidence.

- Observation: …

  Evidence: …

## Decision Log

Record every decision made while working on the plan in the format:

- Decision: …

  Rationale: …

  Date/Author: …

## Outcomes & Retrospective

Summarize outcomes, gaps, and lessons learned at major milestones and at completion. Compare the
result against the original purpose.

## Context and Orientation

Describe the current state relevant to this task as if the reader knows nothing. Name the key files
and modules by full path. Define any non-obvious term you will use. Do not refer to prior plans.

## Plan of Work

Describe, in prose, the sequence of edits and additions. For each edit, name the file and location
(function, module) and what to insert or change. Keep it concrete and concise.

## Concrete Steps

State the exact commands to run, where to run them (i.e. working directory) and any other
required environment. When a command generates output, show a short expected transcript so the
reader can compare. This section must be updated as work proceeds.

## Validation and Acceptance

Describe how to start or exercise the system and what to observe. Phrase acceptance as behavior,
with specific inputs and outputs. If tests are involved, say for example: "run <project’s test
command> and expect <N> passed; the new test <name> fails before the change and passes after>".

## Idempotence and Recovery

If steps can be repeated safely, say so. If a step is risky, provide a safe retry or rollback path.
Keep the environment clean after completion.

## Code Pointers

Add code pointers to both "external" code (e.g. libraries being used, the code being migrated or
rewritten) and the "internal" code (i.e. the code being changed or added to) as a list. Each
list item must include a link (file/path:line_number) and a concise description of what is
located at the link and how it's relevant.

## Artifacts and Notes

Include the most important transcripts, diffs, or snippets as indented examples. Keep them
concise and focused on what proves success.

## Interfaces and Dependencies

Be prescriptive. Name the libraries, modules, and services to use and why. Specify the types,
traits/interfaces, and function signatures that must exist at the end of the milestone. Clearly
state all changes to existing types, and all new types. Prefer stable names and paths such as
`crate::module::function` or `package.submodule.Interface`. E.g.:

In crates/foo/planner.rs, define:
```
    pub trait Planner {
        fn plan(&self, observed: &Observed) -> Vec<Action>;
    }
```
