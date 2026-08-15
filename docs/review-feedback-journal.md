# Review Feedback Decisions Journal

A record of where prompt changes departed from reviewer feedback, and why.

Standing rule: if application code can or should do something about multi-region
resiliency during regional failover between West US 2 and West US, the prompt lists
it — even where feedback asked for its removal. Every such departure is journaled
here with the reviewer's position, the assessment, and the disposition.

## Prompt 14 — Azure Managed Redis

Reviewer strategy: an active-active multi-region cache with West US and West US 2 both
writable and kept in sync through Active Geo-Replication, rather than a single primary
with a read-only geo-replica. West US is the current primary; West US 2 is the new
primary region. Applications write to the Managed Redis instance in their own region.
Three findings are required (own-region endpoint, retry or circuit breaker, regional
Redis health in the health endpoint).

The same strategy states that the design "introduces eventual consistency and
complexity that applications must handle". That sentence, not a departure from
feedback, is the basis for areas 4 through 7 below. The inline annotations that
dismissed several of those areas as infrastructure concerns conflict with the strategy
header in the same feedback cell; where they conflict, the strategy governs.

### Retained on strategy grounds, against inline annotations

**Cross-region consistency assumptions** (was area 6)

* Reviewer annotation: "Infra issue, not App."
* Assessment: the strategy itself assigns eventual-consistency handling to the
  application. The infrastructure supplies the guarantee; whether application code
  assumes something stronger is application code. Session state, distributed locks,
  idempotency keys, and counters written in one region and read in another break
  exactly at failover.
* Disposition: retained as area 4, reframed as a defect in application code rather
  than a replication setting.

**Redis treated strictly as a cache** (was area 9)

* Reviewer annotation: "How is this resliency issue ?"
* Assessment: it is a data-loss issue at regional failover. If the only durable copy
  of state lives in the cache, losing a region loses that state or serves it stale.
  The concern is not durability configuration but whether application code depends on
  the cache as a system of record.
* Disposition: retained as area 5, scoped to state whose only durable copy is in Redis.

**Cache-miss and stale-read handling** (was area 7)

* Reviewer annotation: not marked for removal, but part of the block the reviewer asked
  to align with the strategy, and not one of the three required findings.
* Assessment: the surviving region's cache is cold immediately after failover. Code
  that treats a miss as an error rather than reading the system of record turns a cold
  cache into an outage. No other prompt covers this.
* Disposition: retained as area 6.

**Concurrent multi-region writes** (was area 8)

* Reviewer annotation: not annotated, dropped as part of the same block. Both regions
  are writable by design, so this is a direct consequence of the target architecture.
* Assessment: both regions write their own cache and the caches replicate. Two regions
  writing the same key need an ownership rule or conflict resolution in application
  code; without one the result is silent last-writer-wins.
* Disposition: retained as area 7.

### Removed, agreeing with feedback

**Fallback logic to secondary or tertiary regions** (was area 4)

* Reviewer: "Fallback logic is not part of App, this is with GLB."
* Assessment: agreed. The application writes to its own region only. If the region is
  lost, the global load balancer moves traffic; the application does not reach across
  to another region's cache.

**Clean failback after regional recovery** (was area 11)

* Reviewer: "how is related to app code ? This is infra."
* Assessment: agreed for replication and recovery, which Managed Redis owns. Residual
  application-side risk such as connection pools or DNS entries pinned to a stale
  endpoint is generic client behavior rather than Redis-specific.

### Removed, covered elsewhere

**Application start when Redis is unavailable** (was area 10)

* Reviewer: "Not a multi region resiliency issue."
* Assessment: partly disagree. A failover target that cannot start while its dependency
  is briefly unreachable does block failover. But this is generic startup-failure
  behavior, and `hve-resiliency-researcher-5-1-startup-failure` already emits rows for
  an application that fails to start or reach a healthy state when a dependency times
  out or is partially unavailable.
* Disposition: not duplicated in Prompt 14; coverage relies on Prompt 5.

## Prompt 15 — Azure Storage

**Partial write failure when writing to both regions**

* Reviewer: removed "read and write behavior during regional Storage or service
  failure, including evidenced fallback" as shared-service design.
* Assessment: the strategy has the application writing to both regions. What the
  application does when one of those two writes fails is application code, and the
  consequence is divergence between the regional accounts. The retry and
  circuit-breaker finding covers transient failure, but not the durable-divergence case
  where a remote write is abandoned without compensation, queueing, or a signal.
* Disposition: not yet added. Raised for a follow-up decision because the Prompt 15
  change was already reviewed and committed.

## Prompt 17 — Entra ID

**JWKS retrieval and caching**

* Reviewer: removed as not part of application-code multi-region resiliency.
* Assessment: low confidence. Entra ID is global, so regional failover does not change
  its availability. A missing JWKS cache is a general availability concern rather than
  a regional-failover one.
* Disposition: removal accepted. Recorded so the reasoning is not relitigated.

**Synchronous calls to Entra ID during request handling**

* Reviewer: removed as not part of application-code multi-region resiliency.
* Assessment: low confidence. The surviving region calls the same global endpoint, so
  failover does not materially change latency or availability.
* Disposition: removal accepted.

## Open questions

* The Redis strategy names West US, West US 2, and East US as one replication group,
  while its own goal statement describes the active-active cache as "Azure Managed
  Redis in West US, West US 2" only. The planner context separately forbids referencing
  East US in report output and treats it as a legacy disaster-recovery target outside
  the architecture. Prompt 14 omits East US and assesses the West US 2 to West US pair.
  Confirm which is authoritative.
* Prompt 14 now fixes the topology as active-active on the strategy's authority. This
  is a deliberate exception to the standing rule that non-Kafka prompts assess both
  active-active and active-standby, on the same basis as the Kafka prompts: the
  topology is stated by the platform design rather than left to repository evidence.
* Feedback on Prompt 14 and Prompt 15 flagged content that earlier merged pull requests
  had already removed, including zone-failure scope. The review copy appears to predate
  those merges, so remaining feedback in the same sheet should be checked against
  current files before it is actioned.
