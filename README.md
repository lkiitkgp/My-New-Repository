
# Workflow Parallel Execution Design

## 1. Introduction

This document outlines the design for transitioning the workflow execution engine from a sequential model to a parallel one. This change will significantly improve the performance of workflows by allowing independent blocks to run concurrently.

## 2. Problem Statement

The current workflow execution engine processes blocks sequentially, one after another, as defined in the `executionOrder`. This is inefficient for workflows that contain branches or independent tasks that could be executed in parallel. This sequential processing leads to longer execution times and underutilization of resources.

## 3. Proposed Solution

We propose to re-architect the `executeGraph` function to support the parallel execution of workflow blocks. The new design will be based on a dependency-tracking model, where a block can be executed as soon as all its dependencies are met.

### 3.1. Parallel Execution Logic

The `executeGraph` function will be modified to work as follows:

1.  **Track Completed Blocks:** A set of completed block IDs will be maintained.
2.  **Iterative Execution:** The engine will iterate through the execution order, scheduling all blocks whose dependencies are in the "completed" set.
3.  **Parallel Execution:** All scheduled blocks will be executed in parallel using `Promise.all`.
4.  **Update Completed Set:** As each block completes successfully, its ID will be added to the completed set.
5.  **Loop:** The process will repeat until all blocks in the workflow are in the completed set.

### 3.2. Interruption Handling

Interruptions (e.g., `awaiting-approval`, `awaiting-tool-input`) need to be handled carefully in a parallel environment.

-   When a block is interrupted, the execution of that block will be paused.
-   Other independent blocks will continue to execute in parallel.
-   The overall workflow status will be set to "waiting" until the interruption is resolved.

To manage this, we will introduce a new `InterruptionStatus` enum to provide a clear and robust way to track the state of each interruption.

### 3.3. Resuming from Interruptions

When a user resumes a workflow, it's crucial to identify the correct interruption to process, especially if multiple interruptions have occurred. We will implement a `getLatestInterruption` function that will scan the execution log to find the most recent, unprocessed interruption. This will ensure that the workflow resumes from the correct state.

## 4. Schema Changes

### 4.1. `execution_log.ts`

The `interruption` field in the `IExecutionLog` interface will be updated to use the new `InterruptionStatus` enum. This will provide a more structured and reliable way to manage interruption states.

```typescript
export enum InterruptionStatus {
  AwaitingApproval = 'awaiting-approval',
  AwaitingInputReview = 'awaiting-input-review',
  Approved = 'approved',
  Reviewed = 'reviewed',
  StateEdited = 'state-edited',
  AwaitingStateEdit = 'awaiting-state-edit',
  AwaitingLocalExecution = 'awaiting-local-execution',
  Rejected = 'rejected',
  Resumed = 'resumed',
  AwaitingToolInput = 'awaiting-tool-input',
  ToolInputReceived = 'tool-input-received',
}
```

## 5. API Changes

No breaking changes to the public-facing API are anticipated. The changes will be internal to the workflow execution engine.

### 5.1. Future Breaking Change (Post-AGA)

The current `resumeWorkflow` functionality does not require a `blockId` to identify which interrupted block to resume. In a parallel execution model, it is possible for multiple blocks to be in an interrupted state simultaneously.

To avoid a breaking change before the upcoming AGA release, the new parallel execution engine will have a temporary limitation: **only one block will be allowed to be in an interrupted state at any given time.** This is enforced by checking for existing interruptions before creating a new one.

After the AGA release, we plan to introduce a breaking change to the `resumeWorkflow` API, requiring a `blockId` to be specified. This will allow for the resumption of specific blocks in a workflow with multiple simultaneous interruptions, fully leveraging the capabilities of the parallel execution engine.

## 6. Flow Diagrams

### 6.1. Original Sequential Flow

This diagram illustrates the original workflow execution model. Even with parallel paths in the workflow graph, the blocks are executed one at a time in a predefined sequence.

```
             +---------+
        +----| Block B |----+
        |    +---------+    |
        v                   v
+---------+           +---------+
| Block A |           | Block D |
+---------+           +---------+
        ^                   ^
        |    +---------+    |
        +----| Block C |----+
             +---------+

*Execution*: One block at a time, following a predefined `executionOrder` (e.g., A -> B -> C -> D). Even though B and C could run in parallel, they are executed sequentially.
```

### 6.2. New Parallel Flow (Current Implementation)

This diagram shows the new parallel execution model. Blocks B and C can run concurrently after Block A is complete. However, to avoid a breaking change, only one block can be in an interrupted state at a time.

```
                      +-----------------------+
                 +--->| Block B (Executing)   |---+
                 |    +-----------------------+   |
                 |                                v
+---------+      |                            +---------+
| Block A |------+                            | Block D |
+---------+      |                            +---------+
                 |                                ^
                 |    +-----------------------+   |
                 +--->| Block C (Executing)   |---+
                      +-----------------------+

*Execution*: B and C run in parallel after A completes.
*Limitation*: Only one of B or C can be interrupted (e.g., `awaiting-approval`) at a time.
```

### 6.3. Future Parallel Flow (With Breaking Change)

This diagram represents the future, ideal state after the breaking change. Multiple blocks can be interrupted simultaneously, and the user can resume a specific block by providing its `blockId`.

```
                      +---------------------------------+
                 +--->| Block B (Awaiting Approval)   |---+
                 |    +---------------------------------+   |
                 |                                          v
+---------+      |                                      +---------+
| Block A |------+                                      | Block D |
+---------+      |                                      +---------+
                 |                                          ^
                 |    +---------------------------------+   |
                 +--->| Block C (Awaiting Tool Input) |---+
                      +---------------------------------+

*Execution*: B and C run in parallel.
*Multiple interruptions* are allowed (e.g., B is `awaiting-approval`, C is `awaiting-tool-input`).
*API Change*: `resumeWorkflow` will require a `blockId` to specify which block to resume.
```
