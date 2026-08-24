# Agent Task Enqueued Refresh Design

**Date:** 2026-08-24

## Problem

The scheduled-task list derives each task's run summary from the latest active
or terminal `agent.task` Job. Commit `341e3afb` invalidates that DataApi
projection when execution starts and whenever execution settles, so an open
list observes `running` and every terminal state. A newly persisted Job has no
equivalent Agent-owned notification point. Until it is claimed for execution,
the list can therefore keep showing the previous terminal summary or no summary
instead of `queued`.

This is observable when a Job remains `pending` behind the per-Agent or global
concurrency cap, or when a Job is created as `delayed`. The fix must cover every
`agent.task` enqueue path, including natural schedule fires, Run Now, catch-up,
ordinary enqueue, and transactional enqueue, without importing Agent business
logic into JobManager.

## Goals

- Invalidate the Agent task read model after a new scheduled `agent.task` Job
  has been durably created in `pending` or `delayed` state.
- Cover both `JobManager.enqueue` and `JobManager.enqueueTx` after commit.
- Preserve the existing `341e3afb` notifications for `running` and terminal
  states.
- Keep JobManager generic and keep Agent DataApi endpoint knowledge in the Agent
  domain.
- Ensure notification failures cannot roll back, cancel, or otherwise change a
  successfully enqueued Job.

## Non-goals

- Do not add a generic all-state event stream.
- Do not change Job state transitions, scheduling, dispatch, retry, recovery,
  retention, schemas, or migrations.
- Do not notify for an idempotency hit that returns an already-active Job,
  because no new Job or run-summary transition was created.
- Do not add a notification for `delayed -> pending`; both states map to the
  same `queued` task summary.
- Do not add retry-state handling for `agent.task`; its retry policy has
  `maxAttempts: 1`.
- Do not include startup recovery's `running -> pending` transition. That is a
  separate existing-job recovery concern rather than a new enqueue.

## Existing Boundaries

JobManager owns persistence and dispatch. A schedule fire and catch-up enqueue
occur inside JobManager, so the Agent command service cannot observe all
successful inserts. `JobHandler` already exposes business-owned lifecycle
entry points: `execute` for execution, `onMissed` for schedule misses, and
`onSettled` for terminal processing. The Agent handler uses `execute` and
`onSettled` to publish its current run-summary changes.

AgentTaskService owns the composed task projection and its DataApi invalidation
effects. JobManager must not import AgentTaskService or encode Agent endpoint
paths.

## Alternatives Considered

### Poll the task list in the renderer

This stays entirely in the feature layer and eventually observes every state,
but it is not the requested active-refresh behavior. It adds repeated SQLite
projection queries while the page is open and introduces refresh latency even
though the main process knows exactly when the row commits.

### Notify from AgentJobsService call sites

This can cover Run Now, but natural schedule fires and catch-up enqueue inside
JobManager. Duplicating notification around the visible callers would leave
those paths stale. Passing feature callbacks through schedule registration
would itself add a broader JobManager contract and would need recovery semantics
for persisted schedules.

### Branch on `agent.task` inside JobManager

This could notify immediately but violates the ownership boundary: generic job
infrastructure would depend on AgentTaskService and DataApi projection details.
It would also require another JobManager branch for each business handler that
later owns a composed read model.

### Add a generic JobManager state-change event

A subscriber could filter `agent.task` Jobs and publish projection changes. It
would be more uniform, but it expands the scope to running claims, retries,
delayed promotion, terminal writes, cancellation, and startup recovery. That
would duplicate or replace `341e3afb` and requires a larger state-event
contract than the reported gap needs.

## Design

Add one optional synchronous method to `JobHandler`:

```ts
onEnqueued?(snapshot: JobSnapshot): void
```

The callback means "a new Job row for this handler is durable and visible to
readers." It is a lifecycle boundary, not a general status-change callback.
One callback covers both possible initial states, `pending` and `delayed`.

JobManager invokes it only for a newly created row:

1. `enqueue`: after `jobService.create` succeeds and before dispatch or delayed
   timer arming.
2. `enqueueTx`: in the existing post-commit microtask, after re-reading the
   persisted row and before dispatch or delayed timer arming.

An idempotency match returns the existing handle without invoking the callback.
JobManager catches and logs callback errors, then continues publishing Job
cache state and dispatching/arming normally. A projection notification is a
derived side effect and must not turn a durable Job insert into an enqueue
failure.

`agentTaskJobHandler.onEnqueued` checks `snapshot.scheduleId`. For a scheduled
Job it calls `agentTaskService.notifyReadModelChange([scheduleId])`, matching
the existing running and terminal notification strategy. An ad-hoc Job with no
schedule does nothing because it cannot affect a scheduled-task list item.

## Observable State Coverage

| Run-summary transition | Notification owner |
| --- | --- |
| terminal or no summary -> `queued` through initial `pending` | `onEnqueued` |
| terminal or no summary -> `queued` through initial `delayed` | `onEnqueued` |
| `queued` -> `running` | existing handler `execute` notification |
| `running` -> `completed` / `failed` / `cancelled` | existing handler `onSettled` notification |
| `delayed` -> `pending` | none; projection remains `queued` |

## Error and Ordering Semantics

- The callback never runs before persistence or inside the caller's open
  transaction.
- The callback sees the persisted `JobSnapshot`, including its actual initial
  status and `scheduleId`.
- A callback exception is logged through JobManager's logger and ignored.
- Dispatch does not wait for asynchronous feature work; `onEnqueued` is
  synchronous. The Agent notification API is synchronous as well.
- A fast `pending -> running` transition may publish two invalidations close
  together. DataApi revalidation reads the current database truth, so
  coalescing or observing only `running` is correct.

## Test Strategy

Implementation follows TDD.

1. Add a JobManager contract test whose handler records the persisted snapshot
   observed by `onEnqueued`; verify ordinary immediate enqueue reports
   `pending`, future enqueue reports `delayed`, and no callback runs for an
   idempotency hit.
2. Add a transactional enqueue test proving the callback runs only from the
   post-commit path and receives the persisted row.
3. Add an error-isolation test proving a throwing callback does not make
   `enqueue` fail or prevent the Job from being persisted and dispatched.
4. Add Agent handler tests proving a scheduled enqueued snapshot invalidates
   the task projection and an ad-hoc snapshot does not.
5. Run the focused JobManager and Agent handler tests, then the repository's
   risk-proportionate lint/type checks required for the final code diff.

## Success Criteria

- An open scheduled-task list no longer retains a previous terminal/empty run
  summary after any scheduled `agent.task` Job is durably created.
- Both initial `pending` and initial `delayed` Jobs publish the same `queued`
  projection invalidation.
- Existing running and terminal refresh behavior remains unchanged.
- JobManager remains free of Agent/DataApi endpoint dependencies.
- No persistence, scheduling, retry, or recovery behavior changes.
