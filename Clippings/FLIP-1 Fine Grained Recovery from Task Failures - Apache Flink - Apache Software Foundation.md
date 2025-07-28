---
title: "FLIP-1: Fine Grained Recovery from Task Failures - Apache Flink - Apache Software Foundation"
source: "https://cwiki.apache.org/confluence/display/FLINK/FLIP-1%3A+Fine+Grained+Recovery+from+Task+Failures"
author:
  - "[[Aljoscha Krettek]]"
published:
created: 2025-06-23
description:
tags:
  - "clippings"
---
<table><colgroup><col> <col></colgroup><tbody><tr><th>Discussion thread</th><td><a href="http://apache-flink-mailing-list-archive.1008284.n3.nabble.com/DISCUSS-FLIP-1-Fine-grained-recovery-from-task-failures-td12510.html">http://apache-flink-mailing-list-archive.1008284.n3.nabble.com/DISCUSS-FLIP-1-Fine-grained-recovery-from-task-failures-td12510.html</a></td></tr><tr><th colspan="1">Vote thread</th><td colspan="1"><br></td></tr><tr><th>JIRA</th><td><p><span><a href="https://issues.apache.org/jira/browse/FLINK-4256">FLINK-4256</a> - <span>課題情報を取得中...</span><span><b>ステータス</b></span></span></p></td></tr><tr><th colspan="1">Release</th><td colspan="1">1.9</td></tr></tbody></table>

## Phase 1 (Released in 1.3)

This improvement proposal describes an enhancement that makes recovery more efficient by restarting only what needs to be restarted and building on cached intermediate results.

  

Original Design Document: [https://docs.google.com/document/d/16S584XFzkfFu3MOfVCE0rHZ\_JJgQrQuw9SXpanoMiMo](https://docs.google.com/document/d/16S584XFzkfFu3MOfVCE0rHZ_JJgQrQuw9SXpanoMiMo)

## Motivation

When a task fails during execution, Flink currently resets the entire execution graph and triggers complete re-execution from the last completed checkpoint. This is more expensive than just re-executing the failed tasks.

#### Streaming (DataStream) Jobs

For many streaming jobs, this behavior is not critical, because many tasks have all-to-all dependencies (keyBy, event time) with their predecessors (upstream) or successors (downstream). In that case, operators usually cannot make progress anyways as long as one task is not delivering input or accepting output. Full restart only implies that those tasks also recompute their state, rather than being idle and waiting.

More fine grained recovery can help to reduce the amount of state that needs to be transferred upon recovery. If only 1/100 operators need to recover their state, then the one operator has the full bandwidth to the persistent store of the checkpoints, rather than sharing that bandwidth with the other operators that recover their state.

For some streaming jobs, full restarts are unnecessarily expensive. In particular for embarrassingly parallel jobs (no keyBy() or redistribute() operations), other parallel subtasks/partitions can keep running, and the streaming program as a whole would make progress.

#### Batch (DataSet) Jobs

Batch jobs do not perform any checkpoints and are hence completely restarted in case of a task failure. Batch jobs frequently have all-to-all dependencies between operators, but those are not necessarily pipelined, which makes it conceptually possible to have fine-grained restarts.

  

## Proposed Changes

The core change is to only restart the pipelined connected component of the failed task. This should generalize the failure/recovery model.

We can develop this improvement in two steps:

### Version (1) - Entire connected component is pipelined

That case assumes that all connections between operators are pipelined. The full connected component needs to be restarted.

For jobs that have multiple components (typically embarrassingly parallel jobs) this gives the desired improvement. For jobs with all-to-all dependencies, it will behave like the current failure/recovery model.

| **With Independent pipelines**  ![](https://docs.google.com/drawings/u/1/d/ss0arSUDHVGHk3DaEhCAamQ/image?w=485&h=412&rev=195&ac=1) | **With all-to-all dependencies**  ![](https://docs.google.com/drawings/u/1/d/sYAOJvZ-Hzo4HkhWOZoNgFw/image?w=483&h=410&rev=40&ac=1) |
| --- | --- |

  

### Version (2) - Limit pipelined connected component at intermediate results

To further reduce the amount of tasks that need to be restarted, we can use certain types of data stream exchanges. In the runtime, they are called “intermediate result types”, because each data stream that is exchanged between operators denotes an intermediate result.

#### Caching Intermediate Result

This type of data stream caches all elements since the latest checkpoint, possibly spilling them to disk, if the data exceeds the memory capacity.

When a downstream operator restarts from that checkpoint, it can simply re-read that data stream without requiring the producing operator to restart. Applicable to both batch (bounded) and streaming (unbounded) operations. When no checkpoints are used (batch), it needs to cache all data.

#### Memory-only caching Intermediate Result

Similar to the caching intermediate result, but discards sent data once the memory buffering capacity is exceeded. Acts as a “best effort” helper for recovery, which will bound recovery when checkpoints are frequent enough to hold data in between checkpoints in memory. On the other hand, it comes absolutely for free, it simply used memory that would otherwise not be used anyways.

#### Blocking Intermediate Result

This is applicable only to bounded intermediate results (batch jobs). It means that the consuming operator starts only after the entire bounded result has been produced. This bounds the cancellations/restarts downstream in batch jobs.

  

| ![](https://docs.google.com/drawings/u/1/d/st0-NZqia5aBPwRgAogPlLw/image?w=540&h=442&rev=172&ac=1) |
| --- |

## Public Interfaces

Will affect the way failures are logged and displayed in the web frontend, since failures do not lead the job to holistically go to recovery

The Number-of-restarts parameter or RestartStrategy needs to be interpreted differently

- maximum-per-task-failures or
- *maximum-total-task-failures*

## Compatibility, Deprecation, and Migration Plan

- In the first version, the feature should be selectively activated (StreamExecutionEnvironment.setRecoveryMode(JOB\_LEVEL | TASK\_LEVEL)
- Given the simple impact on user job configuration (and the fact that most users go for infinite restarts for streaming jobs), good documentation of the change should help.

  

## Implementation Plan

Version two strictly builds upon version one - it only takes the intermediate result types into account as backtracking barriers.

### Version (1) - Task breakdown

1. Change ExecutionGraph to not go into “Failing” status upon task failure
2. Add Backtracking and Forward Cancellation. Only one global change (status update beyond a single task execution) may happen in the ExecutionGraph concurrently.

## Rejected Alternatives

*(none yet)*

## Phase 2 (Released in 1.9)

This is a follow-up to Phase 1.

It introduced FailoverStrategies that determine how a job can recover from a task failure.

Existing implementations include:

- RestartAllStrategy, which restarts all vertices
- RestartIndividualStrategy, which only restarts the failed vertex
- RestartPipelinedRegionStrategy, which only restarts the FailoverRegion of the failed task.

From an implementation point-of-view a FailoverRegion is just a collection of vertices that are restarted as a unit. FailoverStrategies can give these regions additional semantics and properties based on how these collections are generated.

The RestartPipelinedRegionStrategy creates these by calculating the weakly connected components of tasks with pipelined result partitions.

Blocking result partitions are usually stored on disk, and as such could conceptually be consumed multiple times without requiring the producing task to be restarted. Note that consuming result partitions multiple times is not supported. (see [Related work](https://docs.google.com/document/d/1YHOpMLdC-dtgjcM-EDn6v-oXgsEQKXSoMjqRcYVbJA8/edit#heading=h.kz6n7qdq31bz)).

## Motivation

The RestartIndividualStrategy can only be used (reliably) in very specific scenarios (where each task is it’s own connected component), but is inherently redundant as the RestartPipelinedRegionStrategy would behave the same.

The RestartPipelinedRegionStrategy can only be used (reliably) in specific scenarios as well, namely in (stateless) streaming jobs where the execution graph consists of multiple dis-jointed graphs (and thus connected components), effectively multiplexing multiple jobs into a single deployment. Only one of these jobs would be restarted on failure.

This strategy is not usable for batch jobs at all as they contain non-pipelined result partitions (see [FLINK-10880](https://issues.apache.org/jira/browse/FLINK-10880)). Since failing consumers have no way to reset or recompute these result partitions they would thus only process part of them, violating at-least-once guarantees.

Finally, jobs with colocation constraints are also not eligible for this strategy.

  

## Proposed Changes

[https://issues.apache.org/jira/browse/FLINK-12068](https://issues.apache.org/jira/browse/FLINK-12068)

On a high-level, this proposal is about extending the RestartPipelinedRegionStrategy to take availability of result partitions into account when deciding which failover regions to restart, potentially restarting failover regions to recompute and/or reprocess result partitions.

### Backtracking logic

Original design document: [https://docs.google.com/document/d/1YHOpMLdC-dtgjcM-EDn6v-oXgsEQKXSoMjqRcYVbJA8](https://docs.google.com/document/d/1YHOpMLdC-dtgjcM-EDn6v-oXgsEQKXSoMjqRcYVbJA8)

Consider the following (simplified) job, consisting of 4 failover regions. All arrows between failover regions represent blocking result partitions.

Let’s assume that **C1** fails for some unspecified reason.

Since FailoverRegions are treated as a coherent unit in terms of failover handling, C1 and C2 at the very least must be rescheduled.

If [partitions are consumable multiple times](https://issues.apache.org/jira/browse/FLINK-12070), and the result partition of B required by C1 is still available, then we do not have to reschedule any other vertices as we can simple reconsume the result partition.

Otherwise, B also must be rescheduled.

Naturally, the same logic must be applied to the input of B, i.e., the result partition of A.

**Special attention however must be given to the D failover region.**

A number of scenarios are imaginable here:

1. D is running, and the required result partition of B is still available,
2. D is running, and the required result partition of B is no longer available,
3. D is finished.

  

In case of 2) we naturally have to reschedule D as well.

In case of 1) and 3) one would instinctively assume that D does not have to rescheduled; after all the data is either still *right there*, or D is finished and shouldn’t consume any data anymore.

Whether this assumption is correct depends on the determinism of B’s output, or rather the output all rescheduled upstream failover regions.

If all regions behave deterministically then the output of B will be the same, and as such the result partitions for D will not change, hence no rescheduling would be required.

However if this is not the case, not rescheduling D can lead to unexpected behavior, inconsistent results and even data loss.

Data loss can occur if the partitioning key used to distributed the output of B is not deterministic. In this case it can happen that data that previously was processed by C1 could now be routed to D1, but this wouldn’t be processed since D1 is still reading the previous version of the result partition of B.

Inconsistent results are easy to imagine considering that the C and D regions work on effectively different data sets, which are generally assumed to be identical (just partitioned).

Since we cannot gauge the determinism ahead of time, by default we *always* have to reschedule D if any upstream failover region is rescheduled, regardless of the state of D.

**In the long-term it will likely be necessary to make this behavior configurable, or create 2 distinct strategies.**

In an implementation that does not reschedule D1 it must be ensured that the system is properly aware of this, for example C must be aware that D1 is not a consumer anymore, and the consumed partitions must not be overwritten nor removed until D1 is finished. This would possibly also require splitting failover regions (i.e. having failover regions not be disjoint graphs), to ensure that a subsequent failure of B does not restart more than is necessary. Overall this would add significant complexity that this should be handled as a follow-up.

With this we end up with the following pseudo-code for the core backtracking logic, which from a given task backtracks upstream towards blocking result partitions, and from there downstream to all consumers.:

  

| // entry point for failover strategies  onTaskFailure(task):  containingRegion = determineFailoverRegion(task)  failoverRegion(containingRegion)  // alternatively return collection of vertices  private failoverRegion(containingRegionregion):  if (!hasRegionBeenScheduled(containingRegionregion)) {  // nothing to do  return;  }  resultPartitions = determineNeededResultPartitions(containingRegion)  for (resultPartition: resultPartitions) {  if (isPartitionStillAvailable(resultPartition)) {  // data still available, so in theory don't have to do anything  // exact details depend on shuffle service implementation and  // whether we can consume data from a TM without  // a task being deployed on it  } else {  producerRegion = getProducerRegion(resultPartition)  failoverRegion(producerRegion)  }  }  reschedule(containingRegion)  // restart all consumer regions that could be affected by this failover  // make behavior configurable?  consumersRegions = getConsumersForRegion(containingRegion)  for (consumerRegion: consumerRegions) {  failoverRegion(consumerRegion)  } |
| --- |

  

### Partition life-cycle management

Original design document: [https://docs.google.com/document/d/13vAJJxfRXAwI4MtO8dux8hHnNMw2Biu5XRrb\_hvGehA](https://docs.google.com/document/d/13vAJJxfRXAwI4MtO8dux8hHnNMw2Biu5XRrb_hvGehA)

Operators in Flink produce output which can be consumed by downstream operators. The collective output of an operator is called the intermediate result.

When executing the operators in parallel the intermediate result is further split up into intermediate result partitions where each parallel sub task of an operator produces an intermediate result partition. The set of all intermediate result partitions forms the intermediate result.

Flink currently supports two types of ResultPartitionType (technically there are more but atm we only need these two):

1. **Pipelined:** The result partition can be directly consumed as soon as data has been produced. The result partition data is kept in memory so that back pressure will be created if too much memory is used. Moreover, the data is not persisted. This partition type is used by streaming and batch applications.
2. **Blocking:** The result partition is first completely produced and persisted before downstream consumers can start reading from it. The current implementation SpillableSubpartition tries to keep data in memory before it spills to disk. This partition type is used by batch applications.

The nice property of blocking result partitions is that they are persisted (usually to disk) from where they can be consumed multiple times. This is beneficial because we can produce a result partition once and let multiple downstream tasks read the same result. Moreover, we can use persisted result partitions for faster recoveries because we don’t have to recompute them.

At the moment, intermediate result partitions are released by the TaskExecutor after they have been consumed once.

In order to enable proper fine grained recovery it is required that blocking result partitions can be consumed multiple times. By having intermediate results persisted one does not need to reschedule the complete topology.

Moreover, by allowing result partitions to out live jobs, it could be possible to share results between different jobs. This could be beneficial for ad-hoc queries as they appear with [interactive programming](https://cwiki.apache.org/confluence/display/FLINK/FLIP-36%3A+Support+Interactive+Programming+in+Flink) ([detailed design document](https://docs.google.com/document/d/17twjcQn70rJnVCXcr74AL44HY3jLeT1leC9rAFsluFg/edit)).

  

In order to make a blocking result partition consumable by multiple downstream operators as well as to use it for recoveries, the decision when to release a blocking result partition needs to be made by the JobMaster which has an overview of the job execution. The JobMaster knows when all consumers of a result partition have terminated and, hence, when the result partition can be released. It also knows when a failover region has been completely executed and, thus, when result partitions are no longer needed for recovery.

In order to avoid that the ResourceManager releases a TaskExecutor which still contains result partitions but no more allocated slots, the TaskExecutors report the set of stored result partitions to the ResourceManager. Only if a TaskExecutor does not contain any result partitions, it can get released. See [FLINK-10941](https://issues.apache.org/jira/browse/FLINK-10941) for more information.

A problem of moving lifecycle management to the JobMaster is what happens with the result partitions if the TaskExecutor loses its connection to the JobMaster? The JobMaster TaskExecutor connection might be interrupted for several reasons: Network problems, JobMaster died, the TaskExecutor died, etc. In all failure scenarios, Flink must make sure that the TaskExecutors don’t amass orphaned result partitions which might fill up local disks up to the point where the TaskExecutor’s machine is no longer usable.

In order to solve this problem, we propose two mechanisms:

1. **Heartbeat based clean up**: Delete all partitions belonging to a job when the connection to the JobMaster times out.
2. **Safety net**: Fail fatally if the TaskExecutor’s disk is full and register a shutdown hook to delete the result partition directory.

#### Heartbeat based clean up

TaskExecutor execute Tasks only as long as they have an open connection to the JobMaster. If the connection times out then all running Tasks belonging to this job get cancelled. Similarly, we propose to do the same for result partitions: As long as the job runs, the JobMaster needs to keep an open connection to all TaskExecutors which have result partitions stored. If the connection is lost, then the TaskExecutor will delete all result partitions belonging to this specific job. This will ensure that there cannot be orphaned result partitions.

Keeping an open connection to a TaskExecutor which has result partitions stored will require changes to when to close the connection on the JobMaster and TaskExecutor side. Concretely, before closing a TaskExecutor connection, the JobMaster

- needs to make sure that it has no more allocated slots from this TaskExecutor
- and that for each result partition ShuffleDeploymentDescriptor.hasLocalResources either returns None or Some(id) with id not being the TaskExecutor’s ResourceID

In the future, we might also introduce a grace period before the result partitions are deleted. That way, the TaskExecutor would have a bit of time to re-register at the JobMaster without losing all produced results. For the moment, we can assume that this grace period is always 0.

## Public Interfaces

- The existing "region" failover strategy will be subsumed by the new failover strategy and renamed to "region-legacy".

## Compatibility, Deprecation, and Migration Plan

The new "region" failover strategy will become the default for batch and streaming jobs.

Users who were not using a restart strategy or have already configured a failover strategy should not be affected.

Streaming users who were not using a failover strategy may be affected if their jobs are embarrassingly parallel or contain multiple independent jobs. In this case, only the failed parallel pipeline or affected jobs will be restarted.

Batch users may be affected if their job contains blocking exchanges (usually happens for shuffles) or the ExecutionMode was set to BATCH or BATCH\_FORCED via the ExecutionConfig.

## Implementation Plan

### Task breakdown

1. [Allow partitions to be consumable multiple times.](https://issues.apache.org/jira/browse/FLINK-12070)
2. [Cache blocking partitions on TaskExecutors and setup a life-cycle.](https://issues.apache.org/jira/browse/FLINK-12069)
3. [Introduce dedicated exception for signaling a missing partition.](https://issues.apache.org/jira/browse/FLINK-6227)
4. [Extend backtracking to stop at Intermediate results that are available for the checkpoint to resume from.](https://issues.apache.org/jira/browse/FLINK-12068)