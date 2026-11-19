# Decisions (ADR-style)

Short records of decisions that shaped the agents. Each one is grounded in the working code; the code itself is not published.

## 1. Rules engine first, LLM as a second opinion

**Context.** Monitoring record review has two kinds of checks: objective ones (timestamps against the operations log, interruption arithmetic, source states) and judgement calls (is this sequence of events biologically plausible, is the narrative consistent with the data). Earlier standalone reviewers covered only the first kind.

**Decision.** Keep the deterministic engine as the first layer and send its findings, with a per-record dossier, to an LLM that must confirm or discard each finding with a reason and a source, and may add its own items classified as Error, Human validation or Suggestion.

**Consequences.** Objective checks stay reproducible and cheap. The model's contribution is visible and auditable item by item. If the model is unavailable, the rule findings are still delivered. The model can be wrong, so its output is never the final word (see 2).

## 2. Agents create drafts; a person sends

**Context.** Review results go to senior reviewers and, indirectly, to vessel teams and the regulator's database. A wrong message is costly and hard to retract.

**Decision.** The drafter writes into a copy of the review-control sheet and creates an Outlook draft. The code has no send path.

**Consequences.** The worst automated failure is a bad draft that a person discards. The price is that a human must open and send every reply, which is acceptable because that review is the point.

## 3. Copy inputs, never modify originals

**Context.** The spreadsheets are regulatory records and macro-enabled workbooks. Opening them in place risks locks, recalculation or accidental saves.

**Decision.** The reviewer copies every input to a temporary directory before parsing. The watcher only reads e-mail. The drafter writes only to a renamed copy.

**Consequences.** Re-running anything is safe. Saved attachments and copies accumulate on disk in per-message folders, so cleanup is a manual task.

## 4. Idempotent dispatch with lock files and seen-sets

**Context.** The watcher runs every five minutes, Outlook can resync old items, a review can take longer than the polling interval, and the same report can arrive twice.

**Decision.** Deduplicate mail by `EntryID` (bounded to the last 3,000 items); deduplicate reviews by a SHA-256 hash of input files; one lock file per agent type with a staleness limit; a "repeat when done" flag instead of concurrent runs; publication only when a report is newer than the published one.

**Consequences.** No duplicate reviews, drafts or publications, and no pile-up of concurrent processes. State lives in small JSON files on one machine, which ties the system to that workstation.

## 5. Two LLM access paths, explicit degradation

**Context.** The reviewer runs on my workstation and on colleagues' machines via an installer. Not every machine has an API key, and keys should not be embedded in a distributed executable.

**Decision.** Use an API key if one is configured locally; otherwise call the signed-in Claude Code CLI in headless mode with no tools, a JSON output schema and an empty working directory; otherwise run rules only. The report states which mode was used.

**Consequences.** No shared secret ships in the installer, and the reviewer still delivers value without any model access. Two code paths must be kept in sync, which output normalisation helps contain.
