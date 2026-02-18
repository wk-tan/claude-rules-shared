# Planning Session

Structured planning mode for designing data platform changes before implementation.

## Trigger

Use this skill when the user asks to:
- Plan a new feature, pipeline, or infrastructure change
- Design a solution before implementing
- Run a planning session or architecture discussion
- Create a plan document for later execution

## Behavior Rules

This skill changes how Claude operates for the duration of the session.

### Do

- Read and use the plan template from [plan-template.md](references/plan-template.md) as the output structure
- Fill in sections **only** as the user provides information — ask for what is missing
- Propose options and trade-offs when there are meaningful alternatives
- Update the plan iteratively as the discussion progresses
- Produce a single, clean markdown plan as the final deliverable

### Do Not

- Execute commands, generate application code, or create infrastructure files
- Fill in sections with assumptions or placeholders the user has not confirmed
- Proceed to implementation — that happens in a separate session with the approved plan
- Skip sections silently — flag them as open questions instead

## Procedure

### Step 1: Load Template

Read `references/plan-template.md` and use it as the skeleton for the plan.

### Step 2: Clarify Objective

Ask the user to describe what they want to achieve. Fill in **§1 Objective** and **§2 Context & Constraints** based on their input. Ask about:
- Target GCP project(s) and environment(s)
- Key services involved
- Upstream/downstream dependencies
- Known constraints or risks

### Step 3: Establish Current and Target State

Fill in **§3 Current State** and **§4 Target State**. If the user is unclear on either, help them articulate it before moving on.

### Step 4: Break Down into Phases

Decompose the work into phases in **§5 Phases & Tasks**. Each phase must have:
- A clear goal
- Concrete tasks with enough detail for implementation
- A **stop-gate** — during implementation, Claude pauses after each phase for the user's go/no-go

### Step 5: Define Validation and Rollback

Fill in **§6 Validation & Testing** and **§7 Rollback Plan**. These are non-optional — every plan must address how to verify success and how to undo changes.

### Step 6: Capture Open Items

Record unresolved decisions in **§8 Open Questions**. The plan is not ready for implementation until all open questions are resolved or explicitly deferred.

### Step 7: Deliver the Plan

Present the completed plan as a markdown file for the user to review.

## Using the Plan in an Implementation Session

When starting a new session to execute an approved plan, the user should:
1. Paste or attach the approved plan
2. Instruct Claude to execute phase by phase, stopping after each phase for review
3. State which resources already exist (to avoid recreation)
4. Specify error-handling preference (stop on first error vs. attempt fix and continue)
5. Name the target GCP project and environment explicitly

## Reference Files

- [plan-template.md](references/plan-template.md) — Plan structure template
