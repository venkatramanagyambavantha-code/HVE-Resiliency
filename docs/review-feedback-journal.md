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

## Prompt 16 — Kafka Active-Standby, and the shared Kafka strategy change

Reviewer strategy: applications connect through DNS rather than region-specific endpoints.
DNS resolution directs traffic to the correct regional topic based on primary and secondary
designation and the health of each region. A minimum Kafka client version of 3.8 is
required. The same "strategy must be explicitly provided rather than inferred" note appears
on this cell as on Active-Active, so both prompts take the identical gate.

### Shared change across both Kafka prompts

Both prompts now declare `${input:kafkaStrategy}` and run only when the supplied value
matches their own strategy. The platform context's Database-to-Kafka Pairing Standard is
demoted from a selector to a cross-check: the database model no longer chooses the prompt,
and a contradiction between the confirmed database model and the supplied strategy is
recorded as a finding rather than changing the strategy or blocking the run. SKILL.md
selects the prompt from the supplied value and asks for it when absent, instead of deriving
it from whether Cosmos DB or Azure SQL was confirmed.

This removed the Tier A and Tier B branching from the Active-Standby prompt, which existed
only to handle a topology established by inference at varying confidence.

### Removed, agreeing with feedback

* Independent regional clusters, active and standby clusters, managed replication,
  mirror-topic state, offset synchronization, promotion, failover, and failback: annotated
  "This will be handled by the kafka shared services setup. We need not asess the App code
  from this perspective." Agreed, and now named as shared-services responsibility that is
  never an application-code finding.
* Failure detection and DNS reset: annotated "Any failure will be detected by the shared
  service team and DNS will be reset as required. No impact on the app." Agreed.
* Zonal readiness: already removed by the first zonal-removal pull request.
* Database-to-Kafka pairing as an eligibility rule: removed, since the strategy is now
  supplied.

The twenty-one concern taxonomy reduces to eleven. Dropped: failback and regional recovery,
cluster-specific credential and trust-store dependencies, West US to West US 2 configuration
symmetry, and the pairing rule. The reviewer asked "What is the source of these? Why do we
need these for resiliency asessment ?", and the honest answer for those four is that they
are platform or deployment concerns rather than application code. The remaining eleven are
all application code.

## Prompt 16 — Kafka Active-Active

Reviewer strategy: the producer always writes to the current region's topic. The consumer
always reads from the current region's topic, and during failover reads from the mirror
topic within the current region. In steady state, reads from the mirror topic are blocked
by a feature flag. Four findings required: producer topic targeting, consumer read source,
the steady-state mirror read control, and Kafka client 3.8 or later.

### Changed at the reviewer's request

**Topology must be declared, not inferred**

* Reviewer annotation: "Kakfa strategy must be explicitly provided rather than inferred. In
  case of wrong detection, entire prompt needs to be rerun. Devs, Architect and App team
  must be aligned on the strategy before asessment is started."
* Disposition: the eligibility gate no longer derives the strategy from the confirmed
  database model. It now requires the Kafka strategy to be explicitly provided as
  Active-Active with the development, architecture, and application teams aligned on it,
  and states that a wrongly detected strategy invalidates the run and forces a full rerun.
  The prompt deliberately does not name who provides the strategy or through what
  mechanism, because the feedback does not say.
* Mechanism: the strategy arrives as a declared `${input:kafkaStrategy}` argument, following
  the `${input:targetDeployment}` precedent on planner-3 and planner-3a. Writing the
  prohibition without a channel would have left an agent with nowhere to read the value,
  and therefore no way to comply except by blocking every run or quietly inferring anyway.
* The Database-to-Kafka Pairing Standard and SKILL.md are changed in the same commit, so
  the framework never infers a strategy and then hands it to a prompt that rejects
  inferred strategies.

### Added, not present in the prompt or the feedback

Eighteen concern groups reduce to seven areas. The four required findings are areas 1
through 4. Three application-code concerns from the original eighteen are retained because
they are consequences of the cutover mechanism the strategy describes, and no other prompt
covers them.

**Duplicate and out-of-order processing across cutover** (was concerns 6, 10, 11, 14, 15)

* Assessment: the strategy's own mechanism creates this. Around a flag transition the
  consumer can read the writable topic and the mirror topic, so the same event can arrive
  twice or out of order. Whether consumption, business workflows, and state transitions are
  idempotent, and whether producer retries can write duplicates, is application code.
* Disposition: retained as area 5.

**Runtime effect of the cutover control** (was concern 16)

* Assessment: finding 3 asks only whether a feature flag exists. If the read source is
  resolved once at startup and cached, flipping the flag does nothing without a restart,
  and the control the strategy depends on silently fails at the moment it is needed.
* Disposition: retained as area 6.

**Broker bootstrap** (was concern 7)

* Assessment: code that persists or hard-codes broker host and port values learned from
  `advertised.listeners` cannot reconnect after regional failover. Application code, and
  not covered by any of the four findings.
* Disposition: retained as area 7.

### Removed, agreeing with feedback

* Reviewer annotation on the concern list: "These are all Infra and setup related. Why App
  code needs to be asessed on these?" Agreed for cluster provisioning, Cluster Linking
  configuration, mirror creation and promotion, cross-cluster offset synchronization,
  global load balancer routing and health probes, and platform failback. These are now
  named in the scope as infrastructure that is never an application-code finding.
* The database-pairing concern is removed as an assessment area, since the topology is now
  declared rather than derived from the database model.
* Zonal readiness: annotated "Cluster will be setup in a way to handle zone failure. Why
  app needs to be zone aware?" Agreed, and already removed by the first zonal-removal pull
  request.

## Prompt 10 — Key Vault

Reviewer strategy: a separate Key Vault per region, values scoped and configured specific to
that region. Key provisioning and cross-region synchronization remain the customer's
responsibility. Three findings required: region-specific vault URL, retry or circuit breaker,
and Key Vault in the application health endpoint.

The reviewer also noted "The current propmt is not explicit on this strategy for the finding
Ask", so the scope section now states the two-vault-per-region model directly rather than
leaving it to be inferred.

### Added, not present in the prompt or the feedback

**Secret retrieval pattern**

* Assessment: the three required findings cover which vault is called, how failures are
  handled, and whether health reflects it, but not how often the application calls Key Vault.
  Fetching secrets on the request path puts Key Vault on the hot path, and refetching on every
  instance start matters specifically at failover, when the surviving region starts or scales
  its instances together and Key Vault throttles. That is application code and is amplified by
  regional failover rather than being a general availability concern.
* Disposition: added as area 4, with a matching query family.

### Removed, agreeing with feedback

* Secret, certificate, and key consistency, failover triggers, platform limitations, and
  synchronization and drift controls: annotated "Not required as part of App Code Asessment".
  Agreed; provisioning and synchronization are the customer's responsibility under the
  strategy.
* Identity and authentication: annotated "Not required for resiliency." Agreed.
* Deployment and infrastructure as code, and pipelines with write guardrails: annotated "Not
  app code resliency requirement. To be addressed as part of DevOps/Infra". Agreed as
  assessment areas.
* Zonal failure: annotated for removal, and already removed by the first zonal-removal pull
  request.

### Retained as an evidence source, not an assessment area

**Deployment and infrastructure as code**

* The reviewer removed it as an assessment topic, which is applied. It is kept in the frozen
  manifest source families, relabelled as the binding site for a regional vault URI, because
  the vault URL that finding 1 depends on is frequently set in a deployment manifest or Helm
  value rather than in source. Dropping it from the manifest would leave finding 1 without
  the evidence it needs.
* Identity and authentication and the pipelines family were dropped from the manifest as well
  as from the areas, since nothing assesses them now.

### Changed

* Health and global load balancer alignment is reframed to whether Key Vault availability is
  reflected in the application health endpoint, matching Prompts 9 and 11 through 15. The
  reviewer asked for it to be "more explicit, if not available add a finding".
* The scope section previously said "Do not assume a two-vault model", which now contradicts
  the strategy that states one vault per region.

## Prompt 11 — AKS and Istio

Reviewer strategy: two independent clusters, each running Istio as its ingress gateway.
All microservices deployed simultaneously on both clusters from the same images,
versions, and configuration. West US 2 node pools spread across three availability
zones; West US has none and compensates with larger node counts. A single Azure
Container Registry geo-replicated to both regions, so each cluster pulls from its local
replica.

### Added, not present in the prompt or the feedback

**Pod statelessness**

* Assessment: state held in the pod rather than externalized is lost when a pod or a
  cluster is lost at failover. The Prompt 13 review removed "state handling through
  stateless pods, externalized sessions, and queues" on the grounds that pod
  statelessness belongs to the AKS prompt; it was not in fact covered there, so that
  removal left a gap.
* Disposition: added as area 11, which makes the Prompt 13 disposition correct.

**Deployment parity, considered and rejected**

* A deployment-parity area was drafted, covering region-pinned image references,
  versions, and configuration, including a container image naming a region-specific
  registry rather than the geo-replicated one.
* Disposition: dropped. Image references, registry selection, and manifest values are
  deployment concerns that application code cannot change. The scope paragraph already
  names registry geo-replication as infrastructure that is never an application-code
  finding, so the boundary is stated without opening an assessment area behind it.

### Changed

**Health-probe alignment**

* The prompt asked "Are health probes aligned between GLB and backend services?", the
  platform-side framing the reviewer rejected on Functions, Storage, Redis, SQL, and
  Cosmos DB.
* Disposition: reframed to whether the application health endpoint reflects the health
  of the dependencies that decide whether the region can serve traffic.

**Question format**

* Reviewer annotation: "Format of file is different from other services."
* Disposition: the ten questions are now declarative assessment areas with an explicit
  instruction to record a finding, matching Prompts 12 through 15, and the
  bounded-evidence protocol now says assessment area rather than question. The wider
  structural difference, that this file has three headings where Storage has thirteen,
  is left for the prompt-structure work.

### Removed, agreeing with feedback

**Zonal failure scope**

* Reviewer annotation: "Zonal failure assessment not required. Focus should be on
  regional one."
* Assessment: agreed, and already removed by the first zonal-removal pull request. The
  strategy's description of availability zones in West US 2 is retained only as
  infrastructure context in the scope paragraph, not as an assessment area.

## Prompt 12 — Cosmos DB

Reviewer strategy: multi-region writes for active-active workloads. Both West US 2 and
West US accept writes simultaneously, eliminating the write region as a single point of
failure.

### Added at the reviewer's request

**`_etag` optimistic concurrency**

* Reviewer annotation: "This is not mentioned and it must be part of a finding in case
  of Cosmos Updates of a document."
* Disposition: added as its own assessment area with its own query family, covering
  update paths that read, modify, and write a document without a precondition. Kept
  separate from the write-safety area so the requirement stays visible.
* Term selection: Microsoft documents `_etag` and `If-Match` optimistic concurrency
  under "Applies to: NoSQL" only, and this prompt is scoped to the Cosmos DB Mongo API,
  where concurrency is expressed through a version or `_etag` value inside an update
  filter. The family carries both vocabularies. These are search terms rather than
  architectural claims: a family is one bundled invocation, so extra terms cost nothing
  and the evidence gates discard false positives, while a missing term is unrecoverable.

### Added, not present in the prompt or the feedback

**Cosmos DB availability in the application health endpoint**

* Assessment: the prompt had "backend health-probe and global load balancer health
  alignment", the platform-side framing the reviewer rejected on Functions, Storage,
  Redis, and SQL. The application-side question is whether its own health endpoint
  reflects Cosmos availability.
* Disposition: reframed, and the query family narrowed to drop global load balancer and
  routing terms.

### Retained against feedback

**Read-your-writes across regions**

* Reviewer annotation: "No custom logic to be implemented. SDK should cover it. No need
  to assess from this front." Also covered by "Point number 4, 6 and 8 are more Infra
  aligned."
* Assessment: partly disagree. The driver carries a session token within one client
  instance, so single-client read-your-writes is covered. It does not carry across
  instances: a load-balanced request that writes through one pod and reads through
  another can miss its own write under session consistency. That is application design.
* Disposition: removed as a standalone area, because the reviewer is right about the
  common case and a dedicated area would mostly produce noise. Recorded here because the
  cross-instance case is real and needs its own area if session consistency is later
  confirmed in evidence.

### Removed, agreeing with feedback

**Mid-request behavior when a write region becomes unavailable** (was area 6)

* Reviewer annotation: failed requests "should be recovered operationally rather than in
  code", since handling it in the application "would introduce excessive complexity".
* Assessment: agreed. With multi-region writes the local region accepts the write, so
  the mid-request window is narrow, and the retry and idempotency areas already cover
  the application-side response.

**Evidence-bound data-loss exposure and no-data-loss acceptance boundary** (was area 8)

* Reviewer annotation: "This should be handled at shared service/infra level. This
  should not be part of App Code."
* Assessment: agreed for Cosmos DB, and deliberately different from the SQL prompt,
  where asynchronous replication leaves an application-visible recovery point.
  Multi-region writes acknowledge locally, so the durability boundary is a platform
  property.

**Failover configuration in the driver** (part of area 2)

* Reviewer annotation: "Failover configuration is part of Infra not app, This needs to
  be removed."
* Assessment: agreed. Driver retries and timeouts stay.

**East US and fallback selection**

* Reviewer annotations: "Why East US? Not able to infer anything from the entire
  statement", and the underlined "East US, hosts, URIs, DNS, and fallback selection"
  marked for removal because failback selection assumes application logic that belongs
  to the global load balancer.
* Assessment: agreed on both. East US is removed from the scope paragraph, the region
  and endpoint selection family, and the priority-derivation sentence. The scope
  paragraph itself was rewritten: it previously instructed the reader to treat
  active-active and multi-region writes as unverified claims, which contradicts the
  strategy that states them as the configuration. That contradiction is why the
  paragraph could not be read coherently.

**Zonal failure**

* Reviewer annotation: "No impcat on application code/design ... This needs to be
  removed."
* Assessment: agreed, and already removed by the first zonal-removal pull request.

## Prompt 13 — Azure SQL

Reviewer strategy: West US 2 is the primary read-write region and West US is the
hot-standby secondary, kept continuously synchronized through asynchronous data
replication so transactional workloads avoid cross-region latency.

The feedback marked the whole assessment list with "All below points are not required
for app code assessment for multi-region resiliency", then annotated individual
entries. Applied literally that leaves Prompt 13 with no assessment areas at all, and
the strategy states no required findings to replace them. The areas below are the
application-code behavior that Azure SQL with Failover Groups and asynchronous
replication still leaves to the application.

### Retained against feedback

**Connection-pool behavior during SQL role changes**

* Reviewer annotation: "Not required multi-region code resliency".
* Assessment: after a Failover Group role change the listener resolves to the new
  primary and pooled connections to the old primary are dead. A pool with a long
  maximum lifetime and no validation can keep handing out dead connections well past
  the failover. That is application configuration, not Failover Group behavior.
* Disposition: retained, merged into the failover-window client behavior area rather
  than standing alone, alongside retry, timeout, and circuit breaker.

### Added, not present in the prompt or the feedback

**Tolerance of the non-zero recovery point**

* Assessment: the strategy specifies asynchronous replication, which means a
  transaction committed at the primary may not have reached the secondary when
  failover occurs. Whether application code assumes a committed transaction survives
  failover, and whether any reconciliation exists for transactions that do not, is
  application code and is the most direct data-loss consequence of the stated design.
  Neither the prompt nor the feedback covered it.
* Disposition: added as an assessment area.

**Read-after-write assumptions against the secondary**

* Assessment: the hot-standby secondary lags the primary by the replication delay.
  Application code that writes to the read-write listener and then reads through the
  read-only listener or the secondary can read stale data. Same class of defect as
  Redis area 4, and likewise application code.
* Disposition: added as an assessment area.

**Azure SQL availability in the application health endpoint**

* Assessment: every other reviewed service prompt now carries this finding, and the
  reviewer asked for it explicitly on Functions, Storage, and Redis. The SQL prompt
  had only "health-probe alignment between the global load balancer and backend
  services", which is the platform-side framing the reviewer rejected elsewhere.
* Disposition: reframed to the application health endpoint, matching Prompts 9, 14,
  and 15.

### Removed, agreeing with feedback

**Split-brain and data corruption during failover**

* Reviewer annotation: "Addressed using SDK. Split brain will not occur as one source
  of write."
* Assessment: agreed. A Failover Group has a single write region, so the application
  cannot create split-brain.

**Write safety including write blocking, fencing, and maintenance mode**

* Reviewer annotation: "Addressed using SDK. Not required multi-region code
  resliency".
* Assessment: agreed. Fencing and write blocking are Failover Group behavior.

**State handling through stateless pods, externalized sessions, and queues**

* Reviewer annotation: "Not required multi-region code resliency".
* Assessment: agreed as a SQL concern. Session externalization is covered by the Redis
  prompt and pod statelessness by the AKS prompt.

**Zonal failure survival**

* Reviewer annotation: "No zonal based assessment".
* Assessment: agreed, and already removed. The zone content left this prompt in the
  first zonal-removal pull request, before this review round.

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
