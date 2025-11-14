# Job Controller State Machine Review - Restart Analysis

## Overview
This document provides a comprehensive review of the Volcano Job Controller state machine, with a focus on understanding all possible roots/causes of job restarts.

## State Machine Structure

### Available States
The job controller implements the following states (defined in `pkg/controllers/job/state/factory.go`):

1. **Pending** (`vcbatch.Pending`) - Job is waiting to start
2. **Running** (`vcbatch.Running`) - Job is actively running
3. **Restarting** (`vcbatch.Restarting`) - Job is being restarted
4. **Terminating** (`vcbatch.Terminating`) - Job is being terminated
5. **Terminated** (`vcbatch.Terminated`) - Job has been terminated
6. **Completing** (`vcbatch.Completing`) - Job is completing
7. **Completed** (`vcbatch.Completed`) - Job has completed successfully
8. **Failed** (`vcbatch.Failed`) - Job has failed
9. **Aborting** (`vcbatch.Aborting`) - Job is being aborted
10. **Aborted** (`vcbatch.Aborted`) - Job has been aborted

## Possible Roots of Restarts

### 1. **Lifecycle Policy-Based Restarts** (Most Common)

#### 1.1 Pod Failure Events (`PodFailedEvent`)
**Location**: `pkg/controllers/job/job_controller_handler.go:254-263`

When a pod transitions to `PodFailed` phase:
- A request is created with `Event: bus.PodFailedEvent`
- The exit code is captured from the container status
- `applyPolicies()` matches this event against job/task lifecycle policies
- If a policy specifies `Action: RestartJobAction` for `PodFailedEvent`, the job restarts

**Trigger Conditions**:
- Pod container exits with non-zero exit code
- Pod fails due to resource constraints (OOMKilled, etc.)
- Pod fails due to node issues

**Code Flow**:
```
updatePod() → PodFailedEvent → applyPolicies() → RestartJobAction → state.Execute()
```

#### 1.2 Pod Eviction Events (`PodEvictedEvent`)
**Location**: `pkg/controllers/job/job_controller_handler.go:339-346`

When a pod is deleted (evicted):
- A request is created with `Event: bus.PodEvictedEvent`
- `applyPolicies()` checks for matching lifecycle policies
- If policy specifies `RestartJobAction` for `PodEvictedEvent`, restart occurs

**Trigger Conditions**:
- Pod is manually deleted
- Pod is evicted by scheduler (preemption, node pressure)
- Pod is evicted by node controller (node not ready, etc.)

#### 1.3 Task Failure Events (`TaskFailedEvent`)
**Location**: `pkg/controllers/job/job_controller_handler.go:269-272`

When a task is detected as failed:
- A request is created with `Event: bus.TaskFailedEvent`
- Triggered when cache detects task failure during pod update

#### 1.4 Any Event (`AnyEvent`)
**Location**: `pkg/controllers/job/job_controller_util.go:202`

If a lifecycle policy specifies `Event: AnyEvent` with `Action: RestartJobAction`:
- **ANY** event (PodFailed, PodEvicted, TaskCompleted, etc.) will trigger restart
- This is a catch-all policy that restarts on any event

#### 1.5 Exit Code-Based Policies
**Location**: `pkg/controllers/job/job_controller_util.go:187-190, 207-210`

Policies can also match on specific exit codes:
- If a pod exits with a specific exit code
- Policy matches: `ExitCode: <code>` with `Action: RestartJobAction`
- Restart is triggered

**Example**:
```yaml
policies:
  - exitCode: 137  # OOMKilled
    action: RestartJob
```

#### 1.6 Job Unknown Event (`JobUnknownEvent`)
**Location**: `pkg/controllers/job/job_controller_handler.go:438-441`

When a PodGroup transitions to `PodGroupUnknown` phase:
- A request is created with `Event: bus.JobUnknownEvent`
- This occurs when tasks become unschedulable (e.g., in gang-scheduling scenarios)
- If a lifecycle policy specifies `RestartJobAction` for `JobUnknownEvent`, restart occurs

**Trigger Conditions**:
- PodGroup becomes unschedulable (part of pods can't be scheduled while some are running)
- Node taints/preemption making job unschedulable
- Resource constraints preventing scheduling

**Note**: There's a TODO comment indicating PodGroup unschedulable event handling could be enhanced.

### 2. **Command-Based Restarts**

**Location**: `pkg/controllers/job/job_controller_handler.go:392-397`

When a `Command` resource is created with `Action: RestartJobAction`:
- Command handler processes the command
- Creates a request with explicit `Action: RestartJobAction`
- This bypasses policy matching and directly triggers restart

**Trigger Conditions**:
- User manually issues a restart command via Command resource
- External system/operator creates Command with RestartJobAction

### 3. **Resume from Aborted State**

**Location**: 
- `pkg/controllers/job/state/aborting.go:31-36`
- `pkg/controllers/job/state/aborted.go:31-36`

When a job is in `Aborting` or `Aborted` state:
- If `ResumeJobAction` is executed
- Job transitions to `Restarting` state
- `RetryCount` is incremented

**Note**: This is technically a resume, but it uses the Restarting state.

## State Transitions Leading to Restart

### Direct Transitions to `Restarting`:

1. **Pending → Restarting**
   - **Trigger**: `RestartJobAction` executed in pending state
   - **Location**: `pkg/controllers/job/state/pending.go:31-36`
   - **Action**: Kills all pods (PodRetainPhaseNone), increments RetryCount

2. **Running → Restarting**
   - **Trigger**: `RestartJobAction` executed in running state
   - **Location**: `pkg/controllers/job/state/running.go:33-38`
   - **Action**: Kills all pods (PodRetainPhaseNone), increments RetryCount

3. **Aborting → Restarting**
   - **Trigger**: `ResumeJobAction` executed in aborting state
   - **Location**: `pkg/controllers/job/state/aborting.go:31-36`
   - **Action**: Kills pods (PodRetainPhaseSoft), increments RetryCount

4. **Aborted → Restarting**
   - **Trigger**: `ResumeJobAction` executed in aborted state
   - **Location**: `pkg/controllers/job/state/aborted.go:31-36`
   - **Action**: Kills pods (PodRetainPhaseSoft), increments RetryCount

## Restarting State Behavior

**Location**: `pkg/controllers/job/state/restarting.go:29-51`

When in `Restarting` state:
1. **Kills all pods** (PodRetainPhaseNone - no pods retained)
2. **Checks retry limit**:
   - If `RetryCount >= MaxRetry`: Transitions to `Failed`
   - Otherwise: Continues restart process
3. **Waits for pods to terminate**:
   - Calculates total replicas across all tasks
   - Checks if `total - Terminating >= MinAvailable`
   - When condition met: Transitions to `Pending` (to restart)
   - Otherwise: Stays in `Restarting` (waiting for termination)

**Key Logic**:
```go
if status.RetryCount >= maxRetry {
    status.State.Phase = vcbatch.Failed
} else if total - status.Terminating >= status.MinAvailable {
    status.State.Phase = vcbatch.Pending
}
```

## Policy Application Logic

**Location**: `pkg/controllers/job/job_controller_util.go:158-214`

The `applyPolicies()` function determines which action to take:

1. **Explicit Action**: If request has `Action` set, use it directly
2. **OutOfSyncEvent**: Always returns `SyncJobAction`
3. **Version Check**: Outdated requests (version < job.Status.Version) → `SyncJobAction`
4. **Task-Level Policies**: Checked first (overrides job-level)
   - Match on event or exit code
5. **Job-Level Policies**: Checked if no task-level match
   - Match on event or exit code
6. **Default**: Returns `SyncJobAction` if no policy matches

**Priority Order**:
1. Request.Action (explicit)
2. Task-level policies
3. Job-level policies
4. Default (SyncJobAction)

## Event Sources

### Pod Events → Requests
**Location**: `pkg/controllers/job/job_controller_handler.go`

| Pod Event | Request Event | Conditions |
|-----------|---------------|------------|
| Pod added | `OutOfSyncEvent` | New pod created |
| Pod updated → Failed | `PodFailedEvent` | Phase changed to Failed |
| Pod updated → Succeeded | `TaskCompletedEvent` | Phase changed to Succeeded AND task completed |
| Pod updated → Pending/Running | `TaskFailedEvent` | Cache detects task failure |
| Pod deleted | `PodEvictedEvent` | Pod deletion detected |

### Job Events → Requests
**Location**: `pkg/controllers/job/job_controller_handler.go`

| Job Event | Request Event |
|-----------|---------------|
| Job added | `OutOfSyncEvent` |
| Job updated | `OutOfSyncEvent` (if spec/phase changed) |

### PodGroup Events → Requests
**Location**: `pkg/controllers/job/job_controller_handler.go:406-446`

| PodGroup Event | Request Event |
|----------------|---------------|
| PodGroup phase → Unknown | `JobUnknownEvent` |

### Command Events → Requests
**Location**: `pkg/controllers/job/job_controller_handler.go:392-397`

| Command | Request Event | Request Action |
|---------|---------------|----------------|
| Command created | `CommandIssuedEvent` | Command.Action |

## Key Code Locations

### State Machine
- **Factory**: `pkg/controllers/job/state/factory.go:62-85`
- **Pending State**: `pkg/controllers/job/state/pending.go:29-62`
- **Running State**: `pkg/controllers/job/state/running.go:31-101`
- **Restarting State**: `pkg/controllers/job/state/restarting.go:29-51`

### Event Handlers
- **Pod Handler**: `pkg/controllers/job/job_controller_handler.go:137-356`
- **Job Handler**: `pkg/controllers/job/job_controller_handler.go:50-135`
- **Command Handler**: `pkg/controllers/job/job_controller_handler.go:368-404`

### Policy Application
- **applyPolicies**: `pkg/controllers/job/job_controller_util.go:158-214`
- **Request Processing**: `pkg/controllers/job/job_controller.go:301-366`

## Summary of Restart Roots

### Direct Causes:
1. ✅ **Lifecycle Policy with PodFailedEvent** → RestartJobAction
2. ✅ **Lifecycle Policy with PodEvictedEvent** → RestartJobAction
3. ✅ **Lifecycle Policy with TaskFailedEvent** → RestartJobAction
4. ✅ **Lifecycle Policy with JobUnknownEvent** → RestartJobAction
5. ✅ **Lifecycle Policy with AnyEvent** → RestartJobAction
6. ✅ **Lifecycle Policy with ExitCode match** → RestartJobAction
7. ✅ **Command with RestartJobAction**
8. ✅ **ResumeJobAction from Aborted/Aborting state**

### Indirect Causes (that trigger events):
- Pod container failures (exit codes, crashes)
- Pod evictions (scheduler, node issues, manual deletion)
- Task failures detected by cache
- Node failures leading to pod failures
- Resource constraints (OOM, disk pressure)
- Preemption by scheduler
- Job becomes unschedulable (PodGroupUnknown) - gang scheduling scenarios
- Node taints preventing pod scheduling

## Recommendations for Debugging Restarts

1. **Check Lifecycle Policies**: Review job/task `spec.policies` for RestartJobAction
2. **Check Pod Events**: Look for PodFailedEvent or PodEvictedEvent in logs
3. **Check Retry Count**: Monitor `status.retryCount` vs `spec.maxRetry`
4. **Check Exit Codes**: If using exit code policies, verify container exit codes
5. **Check Commands**: Look for Command resources with RestartJobAction
6. **Monitor State Transitions**: Track phase changes in job status

## PodGroup Management by Job Controller

The Job Controller creates and manages PodGroups for each Job. PodGroups are used by the Volcano scheduler for gang scheduling (ensuring all pods of a job are scheduled together or not at all).

### PodGroup Lifecycle

#### 1. **PodGroup Creation**

**Location**: `pkg/controllers/job/job_controller_actions.go:641-748`

**When Created**:
- During job initiation (`initiateJob()`) - `pkg/controllers/job/job_controller_actions.go:189`
- During job updates (`initOnJobUpdate()`) - `pkg/controllers/job/job_controller_actions.go:207`
- Called from `syncJob()` when job is being synced

**Naming Convention**:
- PodGroup name: `{job.Name}-{job.UID}`
- Example: If job is `my-job` with UID `abc123`, PodGroup is `my-job-abc123`
- This ensures uniqueness even if jobs are recreated with the same name

**PodGroup Spec Fields** (mapped from Job):
```go
PodGroupSpec{
    MinMember:         job.Spec.MinAvailable,           // Minimum pods required
    MinTaskMember:     minTaskMember,                    // Per-task minimum (from task.MinAvailable or task.Replicas)
    Queue:             job.Spec.Queue,                    // Queue assignment
    MinResources:      calcPGMinResources(job),           // Calculated minimum resources
    PriorityClassName: job.Spec.PriorityClassName,        // Priority class
}
```

**Owner Reference**:
- PodGroup has `OwnerReference` pointing to the Job
- This ensures PodGroup is garbage collected when Job is deleted

#### 2. **PodGroup Updates**

**Location**: `pkg/controllers/job/job_controller_actions.go:702-747`

The controller updates PodGroup when Job spec changes:

**Updated Fields**:
- `PriorityClassName`: Updated if job's PriorityClassName changes
- `MinMember`: Updated if `job.Spec.MinAvailable` changes
- `MinResources`: Updated if calculated resources change
- `MinTaskMember`: Updated if task `MinAvailable` values change

**Update Logic**:
- Only updates if changes are detected (`pgShouldUpdate` flag)
- Compares current PodGroup spec with Job spec
- Updates PodGroup via API server

#### 3. **PodGroup Deletion**

**Location**: `pkg/controllers/job/job_controller_actions.go:153-160`

**When Deleted**:
- When job is deleted (`deleteJob()`)
- PodGroup is deleted by name: `{job.Name}-{job.UID}`
- Errors are logged but `IsNotFound` errors are ignored (idempotent)

#### 4. **PodGroup Event Handling**

**Location**: `pkg/controllers/job/job_controller_handler.go:406-446`

**Event Watcher**:
- Job controller watches PodGroup updates via informer
- Handler: `updatePodGroup()` processes PodGroup status changes

**Event Processing**:
- When PodGroup phase changes to `PodGroupUnknown`:
  - Creates request with `Event: JobUnknownEvent`
  - This can trigger restart if lifecycle policy matches
  - Location: `pkg/controllers/job/job_controller_handler.go:438-441`

**Unschedulable Detection**:
- Checks PodGroup conditions for `PodGroupUnschedulableType`
- Records warning event on Job when unschedulable
- Location: `pkg/controllers/job/job_controller_actions.go:288-293`

#### 5. **Pod-PodGroup Association**

**Location**: `pkg/controllers/job/job_controller_util.go:103-104`

**Pod Annotations**:
- Each pod created by job controller gets annotation:
  - `schedulingv2.KubeGroupNameAnnotationKey = pgName`
  - This links the pod to its PodGroup for gang scheduling
- PodGroup name format: `{job.Name}-{job.UID}`

**Gang Scheduling**:
- Volcano scheduler uses PodGroup to ensure all-or-nothing scheduling
- Pods with same `KubeGroupNameAnnotationKey` are scheduled together
- If not all pods can be scheduled, none are scheduled (prevents partial job execution)

#### 6. **PodGroup Status Monitoring**

**Location**: `pkg/controllers/job/job_controller_actions.go:283-294`

**Status Checks**:
- Controller checks PodGroup phase during `syncJob()`
- If PodGroup phase is not empty and not `Pending`, enables task syncing
- Monitors PodGroup conditions for unschedulable status

**Sync Behavior**:
- If PodGroup is in non-pending phase, controller proceeds with pod creation
- If PodGroup is pending/unknown, may delay pod operations

### PodGroup Controller Relationship

**Note**: There is a separate PodGroup controller (`pkg/controllers/podgroup/`) that:
- Manages PodGroup status based on pod states
- Updates PodGroup phase (Pending, Inqueue, Running, etc.)
- The Job Controller creates/updates PodGroup spec, while PodGroup Controller manages status

**Coordination**:
- Job Controller: Creates/updates PodGroup spec, watches PodGroup status changes
- PodGroup Controller: Updates PodGroup status based on pod states, manages scheduling queue

### Key Code Locations

- **PodGroup Creation**: `pkg/controllers/job/job_controller_actions.go:641-748`
- **PodGroup Deletion**: `pkg/controllers/job/job_controller_actions.go:153-160`
- **PodGroup Event Handler**: `pkg/controllers/job/job_controller_handler.go:406-446`
- **Pod Annotation**: `pkg/controllers/job/job_controller_util.go:103-104`
- **PodGroup Status Check**: `pkg/controllers/job/job_controller_actions.go:283-294`
- **Informer Setup**: `pkg/controllers/job/job_controller.go:206-211`

### PodGroup and Restart Relationship

**JobUnknownEvent Trigger**:
- When PodGroup becomes unschedulable (PodGroupUnknown phase)
- If job has lifecycle policy: `Event: JobUnknownEvent, Action: RestartJobAction`
- Job will restart to retry scheduling

**Example Scenario**:
1. Job is running with all pods scheduled
2. Node taints prevent new pods from being scheduled
3. PodGroup transitions to `PodGroupUnknown`
4. Job controller receives `JobUnknownEvent`
5. If policy matches, job restarts

## Potential Issues/Edge Cases

1. **Race Conditions**: Multiple events can trigger restarts simultaneously
2. **Retry Limit**: Jobs can restart indefinitely if MaxRetry is not set or is very high
3. **AnyEvent Policy**: Can cause unexpected restarts on any event
4. **Version Mismatch**: Outdated requests are ignored, but could cause confusion
5. **Terminating Pods**: Restarting state waits for pods to terminate, which could delay restart
6. **PodGroup Naming**: Old PodGroups (without UID) may exist and need migration
7. **PodGroup Status Lag**: PodGroup status updates may lag behind pod state changes
8. **Gang Scheduling Conflicts**: If PodGroup is unschedulable, entire job is blocked

