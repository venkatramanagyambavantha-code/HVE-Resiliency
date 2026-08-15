---
description: Run bounded Prompt 16 Kafka Active-Standby-Confluent resiliency research
argument-hint: "kafkaStrategy=Active-Active|Active-Standby (this prompt requires Active-Standby)"
---

# HVE Resiliency Researcher 16 Kafka Active-Standby-Confluent

Use [Application Platform Context](../../../instructions/hve-resiliency-platform-context.instructions.md)
as supporting context. Apply every safety-critical control in this prompt directly, regardless of
whether that instructions file is auto-applied.

Execute this self-contained, read-only research workflow to completion. Do not use generic Task Researcher alternatives or rely on nested instruction, skill, or agent resolution.

## Inputs

* `${input:kafkaStrategy}`: (Required) The Kafka strategy for this application, agreed by the development, architecture, and application teams before the assessment starts. Accepted values are `Active-Active` and `Active-Standby`. Match the supplied value case-insensitively after trimming surrounding whitespace, and treat any value outside that pair as unrecognized rather than resolving it by synonym or inference. This prompt runs only for `Active-Standby`; `Active-Active` selects `hve-resiliency-researcher-16-kafka-active-active` instead.

Never infer the Kafka strategy. Do not derive it from the confirmed database model, from the Database-to-Kafka Pairing Standard, or from any repository signal. If `${input:kafkaStrategy}` is absent, ambiguous, or anything other than `Active-Standby`, stop before any discovery action and record the eligibility block in the status-aware artifact.

A wrongly detected strategy invalidates the whole run. If the strategy is later found to be wrong, rerun this prompt in full against the correct strategy rather than amending its findings.

## Scope And Safety Controls

Assess only application code and configuration that produces to or consumes from Kafka under the Confluent Cloud active-standby scenario. Do not assess, redesign, recommend, configure, or provide examples for Confluent provisioning or operations. Do not provide remediation, implementation guidance, configuration examples, or a planning handoff.

Apply these controls directly:

* Scope the repository to application behavior before, during, and after a managed Kafka cluster flip.
* Evaluate the authoritative scenario: regional failover between West US 2 and West US.
* Applications connect through DNS rather than region-specific endpoints. DNS resolution directs traffic to the correct regional topic based on primary and secondary designation and the health of each region.
* Independent regional clusters, the active and standby clusters, managed replication, mirror-topic state, offset synchronization, promotion, failover, and failback are handled by the Kafka shared services setup. They are never application-code findings, and failure detection and DNS reset are the shared service team's responsibility.
* Treat bootstrap selection as application behavior and platform failover orchestration as out of scope.
* Use Apache Kafka client 3.8 or later as the application client threshold only when a prerequisite-confirmed Confluent product or feature supplies an authoritative citation for that claim.
* Produce evidence-only findings with repository-relative file and line citations. Never convert absent external control-plane material into an application finding.
* Keep all tools and repository operations read-only except progressive and final writes to the Prompt 16 output artifact.
* Preserve secrets and personal data through the trusted transient processing rules below.

## Prerequisite Contract

Derive `<repo-name>` from the workspace root as the case-preserving
root-directory basename. Derive `<YYYY-MM-DD>` as the current UTC assessment
date. Read only these repository-prefixed prerequisites; match the `<repo-name>`
segment case-insensitively per the normalization rules below:

* `.copilot-tracking/research/<YYYY-MM-DD>/<repo-name>-hve-resiliency-researcher-1a-research.md`
* `.copilot-tracking/research/<YYYY-MM-DD>/<repo-name>-hve-resiliency-researcher-1b-research.md`
* `.copilot-tracking/research/<YYYY-MM-DD>/<repo-name>-hve-resiliency-researcher-12-cosmosdb-research.md`, when present
* `.copilot-tracking/research/<YYYY-MM-DD>/<repo-name>-hve-resiliency-researcher-13-sql-research.md`, when present

Each producer artifact must begin with YAML frontmatter, optionally preceded by a single `<!-- markdownlint-disable-file -->` HTML comment on the first line only. The frontmatter must contain at least these lowercase keys, each once, with scalar values; additional keys are allowed but ignored:

```yaml
---
title: <any nonempty string>
description: <any nonempty string>
ms.date: <YYYY-MM-DD>
ms.topic: research
source-prompt: <hve-resiliency-researcher-1a|-1b|-12-cosmosdb|-13-sql>
schema-version: <positive integer>
status: current
---
```

`source-prompt` must exactly identify Prompt 1a, 1b, 12, or 13 by matching the corresponding filename infix (`hve-resiliency-researcher-1a`, `hve-resiliency-researcher-1b`, `hve-resiliency-researcher-12-cosmosdb`, or `hve-resiliency-researcher-13-sql`). `status: current` is the producer-completed signal. `ms.date` must match the `<YYYY-MM-DD>` segment of the artifact path, and all consumed artifacts must share the same `ms.date` under exact comparison.

The artifact body must contain, in order, the fixed H2 headings `## Section 1 - Used Azure Services (Evidence Confirmed)` for Prompt 1a or `## Section 1 — Used External Dependencies (Evidence Confirmed)` for Prompt 1b (Prompt 12 and Prompt 13 use their own producer-declared Section 1 heading text), then `## Section 2 - Checked but Not Present`, and `## Section 3 - Not Applicable`; accept either ASCII hyphen `-` or em-dash `—` as the separator, and reject any other H1 or H2. Each dependency entry starts with an H3 whose ordinal-stripped text is its canonical name and is followed by a bulleted list containing at least a name field (`Service name:` in Prompt 1a, `Service / Dependency name:` in Prompt 1b, or the analogous producer-declared name field in Prompt 12 or Prompt 13) and an evidence field (`Evidence:` or `Evidence (binding):`) whose child bullets each contain at least one `<repository-relative-path>:<line>` citation. A citation path uses `/`, is relative to the repository root, contains no `..`, and ends in a positive decimal line number.

Derive per-producer verdicts from the body without inference:

* Prompt 1a and 1b Kafka verdict is `used` when a Section 1 H3 (ordinal-stripped, case-insensitive) contains `Kafka` or identifies an equivalent Kafka-protocol service such as `Event Hubs` with a documented Kafka-protocol endpoint or a Confluent product, and its evidence field carries at least one citation. It is `not-used` when the same canonical Kafka name appears only in Section 2 or Section 3 with a citation. It is `unknown` when neither section presents a citation-backed classification.
* Prompt 12 and 13 database topology verdict is `single-master`, `multi-master`, or `mixed` when a Section 1 H3 (ordinal-stripped) identifies the covered database and a bulleted field labeled `Topology verdict:`, `Topology classification:`, or an equivalent producer-declared topology field states one of those hyphenated values with at least one supporting citation. It is `unknown` in any other case.

Normalize source-prompt names case-insensitively after trimming whitespace. Normalize verdict values to lowercase and the exact hyphenated values above. Do not normalize any other value by synonym or inference.

Apply prerequisite consistency rules before repository discovery:

1. Reject unreadable, malformed, wrong-repository, wrong-producer, nonterminal, or citation-free artifacts as blocked prerequisites.
2. Reject duplicate artifacts for one producer as blocked unless their normalized verdicts and citation sets are byte-for-byte identical after line-ending normalization; retain one and record the duplicate.
3. Kafka use is determined primarily by Prompt 1b, because Prompt 1a is scoped to Azure services and Prompt 1b is scoped to non-Azure services. Continue when the Prompt 1b Kafka verdict is `used`, regardless of the Prompt 1a Kafka verdict, provided the two verdicts do not directly contradict on the same Kafka-protocol backend. A direct contradiction exists only when Prompt 1a Section 1 identifies an Azure Kafka-protocol backend such as Event Hubs with a Kafka-protocol endpoint that Prompt 1b Section 2 or Section 3 disproves for the same broker set with a citation, or vice versa; classify a direct contradiction of that shape as blocked. Classify a Prompt 1b Kafka verdict of `unknown` as blocked. Classify a Prompt 1b Kafka verdict of `not-used` as not applicable, regardless of the Prompt 1a verdict.
4. Take the Kafka strategy from `${input:kafkaStrategy}`. Do not derive it from the confirmed database model, from Prompt 12 or Prompt 13 topology verdicts, or from the Database-to-Kafka Pairing Standard. Classify the run as not applicable and direct the user to `hve-resiliency-researcher-16-kafka-active-active` when the supplied strategy is `Active-Active`; classify it as blocked when no strategy is supplied.
5. Treat the Kafka platform as Confluent Cloud, a confirmed engagement platform fact; do not ask the operator which Kafka provider or Confluent product is in use.
6. Where the confirmed database model contradicts the supplied strategy, for example a multi-master database alongside an `Active-Standby` strategy, record the mismatch as a finding under `KAS-08` rather than changing the strategy or blocking the run.

Do not infer Kafka use, Confluent identity, database service identity, or database topology from generic repository evidence. Stop immediately after rendering the status-aware artifact when the gate is not applicable or blocked. Record the supplied strategy and its evidence pointers in the prerequisite ledger.

## Assumption And Evidence-State Register

Assign every retained assertion exactly one state:

* `repository fact`: proven by a sanitized repository citation
* `prerequisite fact`: proven by a valid prerequisite artifact and its cited source
* `user-confirmed scenario assumption`: explicitly confirmed but not repository-proven
* `external-platform fact`: supported by a minimal authoritative product citation
* `external-platform unknown`: not established by eligible evidence
* `rejected inference`: a tempting conclusion that eligible evidence does not prove

Generic Kafka clients, bootstrap properties, configuration, or a Confluent dependency repository remain generic Kafka evidence. They cannot prove Confluent Cloud or Confluent Platform, Cluster Linking or Replicator, active-standby regions, replication, promotion, failback, synchronized offsets, ACLs, schemas, lag, RPO, overlap, or data loss.

## Conditional External Evidence Protocol

Enter this branch only when a specific platform-behavior assertion needed to validate the causal chain of an otherwise in-scope application finding remains unresolved by repository and prerequisite evidence. When the assertion depends on managed-replication behavior that repository evidence cannot reach, disposition it as `unknown/evidence gap` and select a completed status from bounded repository evidence rather than entering this branch. Never fetch to identify the product or feature, fill missing topology proof, or establish deployed configuration, topology, runtime state, regions, replication health, lag, offsets, ACLs, schemas, RPO, or data loss.

Use only authoritative Confluent vendor documentation for the confirmed product and feature. Access a page only by a claim-specific canonical URL already supplied by eligible evidence or the user; do not use broad web search, search-engine results, recursive link traversal, or links discovered from a fetched page. Enforce a run-wide maximum of two external retrieval attempts or pages and two retained external citations. Count every attempted page, including unavailable or failed retrievals, against the fetch cap. Fetch each canonical URL at most once, cache its sanitized result, and reuse that cache for every later assertion.

For each fetched page, retain only the minimum text needed for the claim. Record `External platform evidence: EXT-### | <source title> | <canonical URL> | retrieved <date>; currentness <published-or-updated indicator, not stated, or stale> | supports: <exact platform-semantics assertion>`. A canonical page with no published or updated indicator may be used only when it is not marked retired, superseded, or version-inapplicable; record `currentness not stated`. External evidence may support only platform semantics for the confirmed product and feature. It cannot supply a repository code location or prove any deployed or runtime fact prohibited above.

Apply these evidence-state transitions without additional discovery:

1. When a fetched authoritative page directly and currently supports the exact assertion, assign that assertion `external-platform fact`, create one `EXT-###` mapping, and map the application-side portion of any finding separately to eligible repository evidence.
2. When the page or canonical URL is unavailable, inapplicable, stale, or lacks enough claim-specific support, assign `external-platform unknown`, disposition the unsupported candidate as `unknown/evidence gap`, and record the exact reason. Select `completed with unknowns/evidence gaps` when this is the only material unresolved application assertion and no higher-precedence status applies.
3. When eligible external sources contradict one another or contradict an attempted inference, retain the minimal contradictory `EXT-###` mappings, assign `rejected inference`, record the contradiction in Rejected Candidates and Unknowns And Evidence Gaps, and do not emit the candidate as a finding. Use `completed with unknowns/evidence gaps` when the contradiction leaves a material assertion unresolved and no higher-precedence status applies.
4. When the external fetch tool is unavailable or fails before the allowed attempt can be evaluated, record an external evidence gap and select `completed bounded partial` as a nonessential tool failure unless a higher-precedence prerequisite, required-tool, artifact, or validation failure requires `blocked prerequisite/tool`. External-tool unavailability alone is never a blocked prerequisite.
5. When the two-attempt or two-citation cap is exhausted before the needed assertion is resolved, stop external access, record the exhausted cap and unexamined assertion, assign `external-platform unknown`, disposition the candidate as `unknown/evidence gap`, and select `completed bounded partial` unless a higher-precedence status applies.

Keep evidence classes mechanically distinct in the artifact and every assertion mapping. Use `Repository evidence: REPO-### | <repository-relative-path>:<exact-line-or-range> | <symbol-or-property> | <minimal sanitized excerpt>` for repository facts. Use `Prerequisite evidence: PRE-### | <producer-artifact>:<exact-line-or-range> | source <repository-relative-path>:<line> | <normalized verdict>` for prerequisite facts. Use the `External platform evidence: EXT-###` format above only in a separate external-source ledger within Assumptions And Evidence States. A finding may reference an `EXT-###` identifier only for its platform-semantics causal link; `Code location (file + line number)` and all application-behavior proof must reference `REPO-###`. Never substitute `PRE-###` or `EXT-###` for repository finding evidence.

## Immutable Production Source Manifest

Construct one immutable text-only manifest before concern searches. Include at most 80 unique files, sorted by repository-relative path, from these roots only:

* Build manifests and resolved dependency reports already present in the repository root
* `src/main/` application source and resources
* `Actionsfile/`, deployment manifests, infrastructure files, container files, and production build or packaging scripts
* Production-bound configuration files directly referenced by an included deployment or runtime binding

Exclude `.git/`, `.copilot-tracking/`, `target/`, generated output, vendored dependencies, binaries, certificates and private-key material, tests, fixtures, samples, documentation, and local or developer profiles unless an included production source explicitly binds them. Include text files only.

Record each manifest item as a default, production-bound value, or unavailable operational value. Apply this effective-value precedence from strongest to weakest: deployment or runtime binding, production profile override, application default, then code default. A stronger source wins only when it explicitly binds the same property. Do not guess unavailable deployment, secret-store, runtime, or external control-plane values.

Follow at most one level of indirection from an included production source. Read no more than 12 indirection targets. Do not mutate the manifest after concern searches begin; newly mentioned out-of-manifest files become evidence gaps.

## Closed Application Concern Taxonomy

Cluster provisioning, Cluster Linking or Replicator configuration, mirror-topic state, promotion, cross-cluster offset synchronization, platform failover detection, DNS reset, and failback are handled by the Kafka shared services setup and are never application-code findings.

Use only the following concern IDs and meanings. Do not add, split, rename, or expand assessment topics. Record a finding wherever the expected behavior is not evidenced.

1. `KAS-01`: DNS-based connection. The application connects through DNS rather than region-specific endpoints. Record a finding where a hard-coded broker, a cached IP address, a region-specific bootstrap setting, or a pinned regional endpoint is evidenced.
2. `KAS-02`: Bootstrap re-resolution. Bootstrap DNS stays authoritative. Record a finding where `advertised.listeners` metadata or learned broker addresses are cached or persisted in a way that bypasses re-resolution, since DNS redirection is how this strategy fails over.
3. `KAS-03`: Direct and transitive Apache Kafka client version 3.8 or later.
4. `KAS-04`: Client recovery across a cluster flip. Producers, consumers, stream processors, admin clients, schema clients, and health checks reconnect with bounded retry backoff, request and delivery timeouts, and metadata refresh.
5. `KAS-05`: Runtime cutover. Record a finding where routing or connection state is resolved once at startup or cached for the process lifetime, so a DNS change has no effect without a restart.
6. `KAS-06`: Duplicate, delayed, and out-of-order processing. Producer idempotence, acknowledgements, delivery callbacks, consumer commit timing, stable group identity, reset policy, replay tolerance, non-idempotent business side effects, outbox or inbox patterns, and commit boundaries. Record a finding where a flip can produce duplicate business outcomes or where strict ordering is assumed.
7. `KAS-07`: Standby freshness. Record a finding where a workflow assumes the standby holds every latest record immediately after the flip, since replication is asynchronous.
8. `KAS-08`: Strategy and database-model consistency. Record a finding where the confirmed database model contradicts the supplied Kafka strategy.
9. `KAS-09`: Backpressure and catch-up after the flip. Bounded queues, poll starvation, max-poll intervals, and memory growth while the application drains a backlog.
10. `KAS-10`: Application lifecycle. Startup, dependency injection, singleton ownership, worker cancellation, graceful shutdown, and readiness.
11. `KAS-11`: Whether Kafka availability is reflected in the application health endpoint.

## Bounded Discovery And Read Protocol

Create one bundled query family for each concern ID using only terms already named by that concern and confirmed repository identifiers. Search each concern family exactly once against the immutable manifest, cache the result, and reuse that cache. Use one additional bundled dependency query for Kafka client declarations and resolved runtime versions. Do not perform broad rediscovery, synonym expansion, or repeated searches.

Enforce these run-wide hard caps:

* 80 manifest files
* 2 discovery rounds
* 24 bundled repository search or query invocations, including manifest construction and dependency resolution
* 60 unique file reads
* 6 corrective rereads total, with at most one corrective reread per file
* 1 indirection level and 12 indirection reads
* 100 retained candidates
* 40 finding rows
* 3 citations per finding row
* 65,536 sanitized retained evidence bytes
* 2 conditional external documentation fetches and 2 external citations total

Round 1 executes the cached query families and creates candidates. Round 2 reads only owning code or configuration needed to verify Round 1 candidates and their assertion relationships. A complete round is saturated when it adds no eligible candidate and no assertion-to-evidence relationship. Stop at saturation, after Round 2, or when any cap is reached, whichever occurs first.

When a threshold is reached, stop new discovery immediately, finish classification from already sanitized retained evidence, record the exact exhausted cap and unexamined manifest or candidates, and use `completed bounded partial` unless terminal-status precedence requires `blocked prerequisite/tool`. Never exceed a cap to complete a finding.

## Trusted Transient Processing

Raw-returning tools may expose source text transiently. Use raw output only to identify and redact sensitive spans. Before retaining, deriving an assessment assertion, quoting, logging, caching, or writing output, replace credentials, passwords, API keys, SASL secrets, JAAS values, certificates, private material, trust-store secrets, tokens, secret-bearing endpoints, and PII with `<redacted:type>`.

Never retain raw tool dumps, secret values, certificate bodies, private keys, or binary content. Count UTF-8 bytes only after sanitization and only for retained excerpts, mappings, candidate records, and coverage records. Discard transient raw output after producing its sanitized representation.

## Candidate And Evidence Controls

Assign candidates stable IDs `CAND-001` onward after sorting by concern ID, repository-relative path, first line, and normalized assertion text. Use the deduplication key `concern ID | scenario | failure mode | owning code path | observable effect`.

Give every candidate exactly one terminal disposition: `finding`, `rejected false positive`, `rejected inference`, `duplicate of <candidate ID>`, or `unknown/evidence gap`. A duplicate may point only to an earlier candidate. Never merge materially distinct failure modes.

Map every substantive assertion to a sanitized citation containing repository-relative path, exact line or line range, symbol or property, and a minimal excerpt. Validate that each cited line exists, the excerpt matches, the evidence state is eligible, and the citation supports the assertion rather than merely mentioning a keyword.

Negative repository claims must name the manifest subset, cached query family, and reads that bound the claim. Coverage citations may support an absence statement in Production Coverage or Unknowns and Evidence Gaps. They cannot serve as finding evidence or justify a priority.

## Priority Classification

Assign priority only from a cited causal chain connecting observed application behavior to an authoritative scenario impact:

* `P0`: Causes outage, data loss, duplicate charges, or inability to fail over safely during regional failure
* `P1`: Materially increases application, data, or customer risk during failure without fully blocking failover
* `P2`: Weakens resilience posture or operational clarity without materially affecting failover correctness
* `P3`: Non-blocking maintainability, readability, duplication, or consistency concern

Do not assign or escalate priority solely from an unknown, rejected inference, external assumption, or missing external-platform evidence. Findings contain no remediation wording.

## Terminal Status

Select exactly one status using this precedence:

1. `blocked prerequisite/tool`: A prerequisite is invalid or a required tool failure prevents prerequisite, manifest, or citation validation.
2. `not applicable`: Valid prerequisites exclude Kafka (Prompt 1b Kafka verdict `not-used`), select a database topology that does not pair with this active-standby prompt (`multi-master`), or classify the Kafka topology directly as Active-Active per Prerequisite Contract Rule 7. In each case, direct the user to `hve-resiliency-researcher-16-kafka-active-active` when the mismatch is a topology mismatch, or to the next applicable service-specific prompt when Kafka itself is not used.
3. `completed bounded partial`: A hard cap or nonessential tool failure leaves declared manifest or candidate coverage incomplete.
4. `completed with unknowns/evidence gaps`: Eligible bounded research completes, but one or more material application assertions remain unknown.
5. `completed with findings`: Eligible bounded research completes with one or more validated findings and no material unknowns.
6. `completed zero findings`: Eligible bounded research completes with no validated findings and no material unknowns.

Artifact write or validation failure changes any otherwise completed status to `blocked prerequisite/tool`. Status selection is deterministic and final.

## Canonical Output Artifact

Write progressively to `.copilot-tracking/research/<repo-name>-hve-resiliency-researcher-16-kafka-active-standby-confluent-research.md`. Write to a temporary sibling file, validate it, then atomically replace the final artifact. On interruption, preserve the latest valid temporary artifact and report its path without presenting it as final.

Use this section order:

1. `Scope And Terminal Status`
2. `Prerequisite Ledger`
3. `Assumptions And Evidence States`
4. `Production Source Manifest And Coverage`
5. `Database-To-Kafka Pairing Verdict`
6. `Findings`
7. `Rejected Candidates`
8. `Unknowns And Evidence Gaps`
9. `Zero Findings Statement`
10. `Validation And Handoff`

The prerequisite ledger records producer, normalized verdict, citations, duplicate or conflict handling, and gate result. The coverage section records budgets consumed, saturation, caps reached, manifest exclusions, query families, reads, and bounded negative claims. The pairing section records `eligible`, `not applicable`, or `blocked` with prerequisite citations. Sections outside Findings carry candidate IDs, rejected inferences, unknowns, and coverage; do not add fields to finding rows.

When status is `completed zero findings`, state that no validated findings were produced within the declared bounded coverage. Otherwise write `Not applicable for this status` in Zero Findings Statement.

## Authoritative Finding Schema

Render every finding with these field names in this exact order and no additional fields:

1. `Issue Description:` Include finding ID, concern ID, and `Regional failover between West US 2 and West US`.
2. `Risk Level (P0/P1/P2/P3):`
3. `Code location (file + line number):`
4. `Why this is a risk to app or regional failover:`
5. `Impact(s) if this is not changed:`
6. `Existing mitigations present (evidence):`
7. `Constraints/limitations (evidence):`

Only `Existing mitigations present (evidence)` and `Constraints/limitations (evidence)` may render `Unknown`, and each such value must reference a corresponding entry in Unknowns And Evidence Gaps. Never use `Unknown` for priority, terminal status, finding or candidate IDs, concern IDs, code locations, citations, prerequisite producers, repository identifiers, scenarios, or required identifiers. If any other finding field cannot be supported, disposition the candidate as `unknown/evidence gap` instead of emitting a finding.

## Validation And Response

Before atomic replacement, verify frontmatter, repository name, exact section order, exactly one terminal status, prerequisite normalization, immutable manifest accounting, all hard-cap totals, candidate dispositions, deduplication keys, finding limits, exact finding field names and order, scenario separation, priority causal chains, citation existence and semantic support, redaction, retained byte count, zero-findings consistency, and absence of recommendations, examples, remediation, or planning content.

Include an HVE next step for `completed with findings`, `completed zero findings`, or `completed with unknowns/evidence gaps`; suggest the next applicable service-specific prompt confirmed by Prompt 1a and 1b, or `/hve-resiliency-researcher-consolidate` when no applicable service remains. In `not applicable` status, direct the user to the sibling prompt implied by the mismatch (for example, `hve-resiliency-researcher-16-kafka-active-active` when the pairing input resolves to `multi-master` or when Prompt 1b classifies Kafka topology as Active-Active) and issue no other HVE next step for that status. Do not include an HVE next step for `blocked prerequisite/tool` or `completed bounded partial`. Mention Prompt 11 only when the HVE sequence and confirmed dependency inventory make it the next applicable service prompt; never route backward, and never suggest it for blocked or bounded-partial status.

Return only:

* Final artifact path, or latest valid temporary path when finalization is blocked
* Terminal status
* Prerequisite gate result and database pairing verdict
* Findings, rejected candidates, and unknown/evidence-gap counts
* Budget consumption and any exhausted cap
* Validation result
* Allowed HVE next step, when status permits

## Output Review

Review every claim against its cited line and semantic mapping. Reconcile contradictions through the declared candidate dispositions and status precedence without additional discovery after saturation or a cap.
