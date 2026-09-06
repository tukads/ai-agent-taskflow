# TaskFlow

## A Minimal File-Backed Coordination Pattern for AI Agents

**Proposed by Tuka · Concept note v0.2 · September 2026**

Your AI session ends. Your work shouldn’t lose its place. TaskFlow describes a lightweight way to preserve task progress, hand off work, and reduce repeated context—without a database or message broker.

TaskFlow separates an agent's temporary conversation from the durable record of its work. A small JSON document records the goal, progress, child tasks and next action. A separate polling process turns changes into human-visible status updates.

**This repository publishes a design concept, not an executable framework or production-ready implementation.** The concept grew from a local agent-workflow experiment. The examples below are illustrative, not a compatibility specification for that implementation.

## The problem

An AI agent may work across multiple sessions, delegate tasks, wait for a person, or stop unexpectedly. Conversation history alone is an inconvenient source of truth for what is complete, what is blocked and what should happen next. Meanwhile, a database, broker and distributed workflow engine may be excessive for a small, single-host personal-agent setup.

## The idea

Keep work state outside the model session. Update it explicitly as work proceeds. Observe it independently to keep the human informed.

1. **One document per flow:** a durable snapshot of the goal, status, revision and child tasks.
2. **One mutation interface:** a CLI validates and serializes writes to that document.
3. **One owner:** an agent session or human coordinates the flow; child workers report results through the same mutation interface.
4. **One polling notification reconciler:** a watcher compares the latest revision with the last successfully reported revision.
5. **Explicit session rehydration:** a replacement session reads unfinished flows and decides how to continue.

The core has no database or message-broker requirement. Filesystem semantics, a scheduler and any selected agent or notification services remain dependencies.

## Benefits

- **Continuity across sessions:** A new session can read the recorded goal, completed work, blockers and next action instead of reconstructing everything from chat history. Resuming execution remains an explicit decision.
- **Clear progress visibility:** People can see what is running, waiting or finished without repeatedly asking the agent for status.
- **Low infrastructure overhead:** For small, single-host workflows, the core avoids operating a separate database or message broker.
- **Inspectable and portable records:** Plain JSON is easy to read, back up, compare and transfer with ordinary filesystem tools. Moving records does not automatically move their referenced artifacts or runtime dependencies.
- **Better handoffs:** A persistent owner record and linked child tasks make responsibilities, outstanding work and result locations explicit.
- **Potentially lower token usage and cost:** A new session can load a compact task snapshot and selected results instead of replaying the full conversation. Reusing saved findings can avoid repeated analysis, while a deterministic watcher can report progress without an LLM call for every status update. Savings depend on the integration actually selecting concise context; storing JSON alone does not reduce the context sent to a model. State updates and handoffs also add overhead. Lower token usage reduces monetary cost only where billing is usage-based; subscription plans may instead benefit through reduced allowance consumption.
- **Independent notifications:** The watcher can report the latest saved progress even when the conversational session is unavailable; it does not infer unrecorded progress or restart stalled work.
- **Controlled notification volume:** Coalescing revisions and spacing updates lets the human follow meaningful changes without receiving every small edit.
- **Model and channel flexibility:** The concept separates work tracking from execution and delivery, allowing different agent runtimes or notification channels through suitable adapters.

The main benefit is **a small, understandable continuity layer between temporary agent sessions and long-running human goals**. These are intended design benefits, not measured performance claims or distributed-execution guarantees.

## Architecture

```text
Human request
     |
     v
Owner agent/session -------> Optional child worker
     |                            |
     +-------- updates/results ---+
     |
     v
Validated CLI mutation interface
     |
     v
Flow JSON snapshots <---------- New session reads open work
     |
     | periodically read
     v
Notification watcher <------ Delivery cursor / last reported revision
     |
     v
Human-facing status channel

Terminal flow -> archive -> retained for final notification / inspection
```

The watcher is an observer, not the task executor. Model selection and fallback are optional execution-adapter concerns, not prerequisites of the pattern.

## Illustrative state

```json
{
  "schema_version": 1,
  "id": "flow-example-001",
  "goal": "Compare three data-platform architecture options",
  "status": "running",
  "revision": 4,
  "owner": "architecture-session",
  "updated_at": "2026-09-06T10:00:00Z",
  "progress": "Requirements captured; comparison in progress",
  "next_action": "Review child findings and draft a recommendation",
  "children": [
    {
      "id": "task-001",
      "goal": "Compare operating constraints",
      "status": "done",
      "result_ref": "artifacts/constraints.md"
    },
    {
      "id": "task-002",
      "goal": "Assess integration options",
      "status": "running"
    }
  ]
}
```

A parent with children forms a tree. Supporting arbitrary dependency graphs requires explicit dependency edges, cycle checks and readiness rules; it does not follow automatically from child-task records.

## Lifecycle

Suggested states are `pending`, `running`, `waiting`, `done` and `failed`.

- Creation records the goal before work begins.
- Each meaningful update advances the revision and records a useful next action.
- Waiting includes a reason and the condition needed to continue.
- Completion records a result reference; failure records a reason and any recoverable work.
- Terminal records move out of the active directory into an archive.

Reopening a terminal flow should be an explicit operation or a new linked attempt, rather than silently changing its meaning.

## Notification reconciliation

A timer can run the watcher every two minutes. A minimum interval, such as 110 seconds between updates to one status message, limits notification frequency. These timings are deployment choices, not essential properties.

The watcher compares flow revisions against separate delivery metadata. If a newer revision is eligible for notification, it sends or edits the status message and persists the successful delivery cursor. Intermediate progress revisions may be coalesced.

Terminal notifications must not disappear merely because a flow was archived. One design is to scan both active records and terminal records whose final revision has not been acknowledged. A single watcher instance or lock prevents overlapping timer runs.

This is reconciliation of *reported progress*, not Kubernetes-style convergence of an application's desired operational state.

## Persistence and recovery boundaries

The record survives a process restart only if it was successfully persisted and the underlying storage survives. It tells a new session what was last recorded; it does not restore an interrupted model call, shell process or network request.

After restart, the owner should inspect stale running tasks, check available result artifacts and reconcile actual outcomes before choosing to continue, retry or fail. An operation may have completed externally before its result was saved. Blind retry can repeat side effects.

Recommended implementation measures, not guarantees supplied by this concept alone:

- Write a temporary file and atomically replace the snapshot on the same filesystem.
- For power-loss durability, consider file and directory synchronization as supported by the platform.
- Serialize read-check-write operations with a lock. Revision checks without an atomic critical section do not prevent lost updates.
- Validate schemas and preserve corrupt records for diagnosis rather than discarding them.
- Keep delivery metadata separate from authoritative work state.
- Use stable identifiers and idempotency mechanisms where external systems support them.
- Retain backups; atomic replacement is not protection against disk loss.

A mutable snapshot is **not a write-ahead log**, an event history or a replay engine. A send can succeed before its delivery cursor is saved, so exactly-once notification delivery is not implied either.

## Where it fits

- Personal or small-team agent workflows on one host.
- Low-volume research, document preparation and architecture assessments.
- Tasks where inspectable state and clear handoff matter more than automated scheduling.
- Work where a human or owner agent can reconcile uncertain outcomes.

## What it deliberately does not provide

- A distributed task queue or worker-lease protocol.
- Automatic durable execution or deterministic replay.
- Exactly-once side effects.
- General DAG scheduling, dependency resolution or retry orchestration.
- High-availability storage, multi-tenant isolation or distributed transactions.

If those are requirements, evaluate an established workflow engine or queue rather than assuming file snapshots supply them.

## Related patterns and comparison

This concept was developed independently from a practical agent-workflow need. The references below were added afterward for comparison; they were not sources used to develop the idea.

- [Kubernetes controllers](https://kubernetes.io/docs/concepts/architecture/controller/) illustrate observation-and-action control loops. TaskFlow narrows the observer's responsibility to reporting recorded progress.
- [Temporal Workflow Execution](https://docs.temporal.io/workflow-execution) illustrates a substantially richer durable execution model. TaskFlow's snapshots are not an equivalent execution mechanism.

The contribution proposed here is a deliberately small composition for agent-session continuity: explicit work snapshots, linked tasks, an independent notification observer and a clear recovery handoff, without requiring a broker or database.

## Open design questions

- How should ownership transfer between sessions be recorded?
- When should stale tasks be flagged, and who decides whether to retry?
- How much event history is worth retaining alongside snapshots?
- Which minimal fields enable portable handoffs between agent runtimes?
- When does operational complexity justify migration to a workflow engine?
