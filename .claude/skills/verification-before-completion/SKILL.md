# Verification Before Completion

Evidence before claims, always. Requires running verification commands and confirming output before making any success claims.

## Trigger

Use this skill when about to:
- Claim work is complete
- Claim a phase is done
- Claim a fix is applied
- Claim a deployment has succeeded

## Iron Law

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

If you haven't run the verification command **in this message**, you cannot claim it passes.

## Gate Function

Every completion claim must pass through all five steps in order. Skip any step and the claim is **unverified**.

### 1. IDENTIFY

What command proves this claim?

### 2. RUN

Execute the full command. Fresh — not from a previous message or cached output.

### 3. READ

Read the complete output. Check the exit code.

### 4. VERIFY

Does the output confirm the claim?

- **No** → State the actual status with evidence. Do not proceed.
- **Yes** → Continue to step 5.

### 5. CLAIM

Only now may you state the claim, citing the evidence from step 3.

---

## Common Failures

| Claim | Requires | Not Sufficient |
|-------|----------|----------------|
| Tests pass | Test command output with 0 failures | "should pass" |
| Linter clean | Linter output with 0 errors | Partial check |
| Build succeeds | Build command exit 0 | "linter passed" |
| Bug fixed | Reproduce original symptom, verify gone | "code changed" |
| Phase complete | All phase-scoped §7 checks executed | "tasks done" |
| Cloud Run deployed | `gcloud run services describe` showing expected revision | "deploy command ran" |
| BigQuery table updated | `bq show` or query result | "Dataform completed" |
| Pub/Sub message delivered | Subscription pull or delivery logs | "publish succeeded" |
| Dataform execution succeeded | Execution log showing all assertions passed | "no errors in terminal" |

## Red Flags

Phrases that signal an unverified claim — if you catch yourself saying these, **stop and run the verification**:

- "should work now"
- "looks correct"
- "I'm confident"
- "seems to"
- "probably fine"
- Expressing satisfaction before verification: "Great!", "Done!", "Perfect!"

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "Should work now" | Run the verification. |
| "I'm confident" | Confidence ≠ evidence. |
| "Linter passed so build must pass" | Linter ≠ build ≠ deploy. |
| "It's a trivial change" | Trivial changes break production. |
| "I already checked manually" | Show the output. |
| "Just this once" | No exceptions. |

## When to Apply

Before **any** of the following — non-negotiable:

- Completion claim
- Positive statement about work state
- Commit
- Phase sign-off
- Moving to next task
