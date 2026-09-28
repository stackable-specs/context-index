---
id: context-index
layer: architecture
extends: []
---

# Context Index

**Status:** Draft, revision 2 (2026-09-28). Revision 2 adds judgment services with Jev (§7.12). Not implemented. Requirements are written in EARS. Requirements that rest on an unresolved decision are marked **(OQ-n)**; see §10.

## 1. Purpose

AI agents need organizational context that is spread across Jira, GitHub, Google Drive, Zoom, Slack and other systems. Giving every agent credentials to every system is expensive to govern, and each agent rebuilds the same picture from scratch.

This spec defines the **Context Index**: a verified, OKF v0.2 knowledge bundle kept up to date from a **context stream** of organizational events. Agents read the stream through a filter set by their role. Curator agents propose knowledge from raw events. Users' agents, and people when a policy requires it, verify the proposals. Only verified changes are committed to the index. Worker agents rely on the committed index, which can be stored in GitHub, GitLab, Confluence, Google Drive, S3 or a similar store.

## 2. Scope

**In scope**

- The event envelope and the event types the system uses.
- Role-scoped access to the stream.
- Proposing, verifying and committing index changes.
- The verification policy.
- The content rules for the OKF bundle.
- Freshness, traceability, and writing the index to external stores.

**Out of scope**

- The choice of event store, broker, connector tooling or models. Earlier design work used TigerData and n8n; this spec is vendor-neutral.
- The user interface of a context node beyond what verification requires.
- Semantic search or embeddings over the index.
- Attested Computation concepts (OKF §10), which a later revision may use for governed projections.

## 3. References

- **external** [Open Knowledge Format v0.2](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md): bundle format, `sources`, `generated`, `verified`, trust tiers, `index.md`, `log.md`.
- **internal** `knowledge/sources/the-context-stream-a-new-architecture-for-enterprise-ai.md`: the Context Stream architecture.
- **internal** `knowledge/sources/from-context-streams-to-causal-memory.md`: derived events and causal chains.
- **internal** `chatgpt/projects/centralized-context-stream.md`: the owner's design conversations, including the Context Mesh implementation spec (context nodes, user-bound credentials, interpreter quorums, single DRI approver).
- **external** [TypeSafe documentation](https://docs.typesafe.ai/llms.txt): Jev, the Choice, Noul and Score primitives, confidence, and the citation-check, entity-alignment, reranking and confidence-routing recipes.
- **external** [Jev on OpenRouter](https://openrouter.ai/typesafe/jev-1.13): model page, limits, pricing and data policy.
- **internal** `docs/adr/006-adopt-typesafe-jev-skill-via-openrouter.md`: ADR-006, Jev only through OpenRouter, pinned to `typesafe/jev-1.13`.
- **internal** Context Index explainer page: https://claude.ai/artifact/KK48ouQ9pK1rpvfWov7KEi

## 4. Terms

EARS requires each system name to be used the same way every time. This spec uses the names below exactly as written.

| Name | Meaning |
|------|---------|
| **the stream** | The append-only, replayable log of events for one organization. |
| **the context node** | The runtime on one user's machine or account. It runs that user's agents with that user's credentials and role, and shows verification requests to the user. |
| **the curator agent** | An agent on a context node that reads raw events and proposes index changes. |
| **the verifier agent** | An agent on a context node that checks proposals and votes. |
| **the worker agent** | An agent on a context node that does work using the committed index. |
| **the committer** | The single process that applies verified proposals to the index and publishes commit events. |
| **the index publisher** | The component that writes committed index content to the primary store and any mirrors. |
| **the judgment service** | The component that asks Jev narrow, typed questions and returns typed answers with probabilities and confidence. It runs beside the stream, the committer and each context node, and never writes content. |
| **Jev** | TypeSafe's System One model, called through OpenRouter as `typesafe/jev-1.13`. It answers Choice, Noul (yes/no probability) and Score questions; it does not generate text. |
| **claim check** | The judgment service's check of each footnoted claim in a proposal against the event the footnote cites. |
| **generative model** | A text-generating model, such as Claude, that curator agents use to write proposed content. |
| **the index** | The OKF v0.2 bundle of verified concepts. |
| **the verification policy** | The `Policy` concept in the index that maps each kind of change to the verification it needs. |
| **source system** | An external system (Jira, GitHub, Zoom, Drive, Slack, a CRM) whose changes enter the stream through a connector. |
| **role scope** | The set of event kinds and source scopes a role may read. |
| **DRI** | The directly responsible individual named by a policy rule as the human approver. |

## 5. Roles

| Role | Reads | Writes |
|------|-------|--------|
| Source system (via connector) | Nothing | Raw events |
| Curator agent | Raw events in its role scope, the index, open proposals | `index.change.proposed` |
| Verifier agent | Proposals in its role scope and the events they cite | `verification.vote` |
| Human approver (through their context node) | Verification requests addressed to them | `verification.approved`, `verification.rejected` |
| Committer | All proposals and verification events, the verification policy | The index (through the index publisher), `index.change.*` events, `verification.requested`, `verification.checked` |
| Worker agent | `index.change.committed` events and the index | Work in source systems, which returns as raw events |
| Judgment service | The state its caller passes in | Typed answers to its caller. The committer records claim checks as `verification.checked`; the stream records sensitivity results as `event.scope.restricted` |

## 6. Event model

### 6.1 Envelope

Every event carries these fields.

| Field | Meaning |
|-------|---------|
| `offset` | Unique, increasing position in the stream. |
| `address` | `stream://<org>/events/<offset>`. |
| `time` | ISO 8601 datetime with a UTC offset. |
| `type` | One of the types in §6.2, or a raw event type named by its connector (for example `github.pr.merged`). |
| `kind` | `raw`, `proposal`, `verification` or `index`. |
| `actor` | An OKF actor: `<producer>/<version>`, `human:<id>` or `process:<id>`. |
| `on_behalf_of` | The user an agent acted for. Empty for source systems and processes. |
| `source_scope` | For raw events, the scope used by role filters (for example `sales`, `engineering`). |
| `refs` | Offsets of the events this event derives from or responds to. |
| `payload` | Type-specific content. |
| `schema_version` | The envelope version. |

### 6.2 Event types defined by this spec

| Type | Kind | Published by | Payload |
|------|------|--------------|---------|
| `index.change.proposed` | proposal | Curator agent | Target path, change type (`create`, `update`, `deprecate`), kind of change, base commit, proposed file content |
| `verification.vote` | verification | Verifier agent | `agree` or `disagree`, reason |
| `verification.requested` | verification | Committer | Proposal offset, approver |
| `verification.approved` | verification | Context node (human actor) | Proposal offset, comment |
| `verification.rejected` | verification | Context node (human actor) | Proposal offset, reason |
| `verification.checked` | verification | Committer (process actor) | Check name, result, evidence |
| `index.change.committed` | index | Committer | Target path, commit offsets counted, store revision |
| `index.change.rejected` | index | Committer | Proposal offset, reasons |
| `index.change.expired` | index | Committer | Proposal offset, rule, missing verifications |
| `index.change.conflicted` | index | Committer | Proposal offset, current commit of the target |
| `index.concept.stale` | index | Committer | Concept path, the newer raw event |
| `stream.access.denied` | index | Stream | Agent, user, requested scope |
| `event.scope.restricted` | index | Stream | Restricted event offset, sensitive category, probability, judgment record |

Every event whose outcome depends on a Jev answer carries a **judgment record** in its payload: the questions asked, the answers with their probabilities and confidence, the OpenRouter generation `id`, the dated model ID, and `usage.cost`.

## 7. Requirements

### 7.1 Stream (STR)

- **STR-001** The stream shall store every event as an immutable, append-only record.
- **STR-002** The stream shall assign each event a unique offset that is greater than the offset of every earlier event.
- **STR-003** The stream shall give each event the address `stream://<org>/events/<offset>`.
- **STR-004** When a source system reports a change, the stream shall append the change as a raw event whose `actor` identifies the source system.
- **STR-005** When a subscriber requests replay from an offset, the stream shall deliver, in offset order, every event from that offset that the subscriber's role scope permits.
- **STR-006** If an event is missing a required envelope field, then the stream shall reject the event with the name of the missing field.
- **STR-007** If an event's `refs` contains an offset that does not exist, then the stream shall reject the event.
- **STR-008** If a producer requests a change to or deletion of an appended event, then the stream shall reject the request.
- **STR-009** When a published event is found to be wrong, the stream shall accept the correction as a new compensating event whose `refs` includes the original event.

### 7.2 Role-scoped access (ACC)

- **ACC-001** The stream shall associate every subscription with exactly one user and one role.
- **ACC-002** The stream shall deliver to each subscription only the events that its role scope permits.
- **ACC-003** The stream shall deliver a raw event to an agent only when the agent's user holds access to the event's source system.
- **ACC-004** While an agent holds the worker role, the stream shall deliver only `index.change.committed` and `index.concept.stale` events to that agent.
- **ACC-005** The context node shall run each agent with the credentials and role of the node's user.
- **ACC-006** When a user's access to a source system is revoked, the stream shall stop delivering that system's raw events to every agent acting for that user.
- **ACC-007** If an agent requests events outside its role scope, then the stream shall deny the request and publish a `stream.access.denied` event.

### 7.3 Curation (CUR)

- **CUR-001** When a curator agent finds information in a raw event that the index does not hold, the curator agent shall publish an `index.change.proposed` event.
- **CUR-002** The curator agent shall include in each proposal the target path, the change type, the kind of change from the verification policy, the base commit of the target concept, and the full proposed file content.
- **CUR-003** The curator agent shall list in each proposal's `refs` every event the proposed content cites.
- **CUR-004** The curator agent shall cite only events that its own role scope permits.
- **CUR-005** When a curator agent prepares a proposal, the curator agent shall search the index for a concept about the same entity.
- **CUR-006** If the index already holds a concept about the same entity, then the curator agent shall propose an update to that concept.
- **CUR-007** If an open proposal already targets the same concept with the same claim, then the curator agent shall publish a `verification.vote` on that proposal.
- **CUR-008** When a curator agent receives an `index.concept.stale` event in its role scope, the curator agent shall evaluate the newer raw event for a proposed update.
- **CUR-009** When a curator agent receives an `index.change.rejected` event for its own proposal, the curator agent shall record the rejection reasons for use in later proposals.

### 7.4 Verification policy (POL)

- **POL-001** The committer shall read the verification policy from the `Policy` concept at `policies/verification.md` in the index.
- **POL-002** The verification policy shall define, for each kind of change: whether a claim check must pass, the minimum number of agreeing agents, whether the agents must act for different users, whether the proposer counts, the named human approver if one is needed, the deterministic check if one is allowed, and an expiry period.
- **POL-007** The verification policy shall define the thresholds the judgment service's callers use: triage, concept match, claim check, routing, sensitivity and guardrail.
- **POL-003** The committer shall evaluate each proposal against the policy version committed at the proposal's offset.
- **POL-004** If a proposal's kind of change matches no rule in the verification policy, then the committer shall require approval from the owner of the verification policy.
- **POL-005** When a proposal targets the verification policy, the committer shall require approval by the named human approvers of the policy.
- **POL-006** If a proposal targets the verification policy and has no `verification.approved` event from a named human approver, then the committer shall leave the policy unchanged.

### 7.5 Verification (VER)

- **VER-001** When a proposal is published, the committer shall route it to verifier agents whose users' role scopes permit every event the proposal cites.
- **VER-002** When a verifier agent receives a proposal, the verifier agent shall check each claim in the proposed content against the events cited for it.
- **VER-003** The verifier agent shall publish its result as a `verification.vote` event with the value `agree` or `disagree` and a reason.
- **VER-004** The committer shall count at most one vote per user for each proposal.
- **VER-005** Where a rule requires agents that act for different users, the committer shall count only votes whose `on_behalf_of` users differ from each other and, unless the rule counts the proposer, from the proposer's user.
- **VER-006** Where a rule requires a human approver, when the rule's agent votes are satisfied, the committer shall publish a `verification.requested` event addressed to the approver.
- **VER-007** When a `verification.requested` event reaches the approver's context node, the context node shall show the approver the proposed content, the cited events and the votes.
- **VER-008** When the approver decides on a request, the context node shall publish a `verification.approved` or `verification.rejected` event with the actor `human:<approver id>`.
- **VER-009** The context node shall publish an event with a `human:` actor only in response to an explicit action by that person.
- **VER-010** Where a rule allows a deterministic check, when a proposal of that kind is published, the committer shall run the check and publish a `verification.checked` event with a `process:` actor.
- **VER-011** If a proposal receives a `disagree` vote, then the committer shall escalate the proposal to the rule's human approver, or to the owner of the target concept when the rule names no approver. **(OQ-3)**
- **VER-012** The committer shall count a claim check as a verification check, never as an agent vote.
- **VER-013** Where a rule requires agents that act for different users, the committer shall count as one vote all votes whose agents used the same model on the same cited events. **(OQ-9)**

### 7.6 Commit (COM)

- **COM-001** The committer shall be the only component that changes the index.
- **COM-002** When a proposal satisfies its rule, the committer shall apply the proposed content to the index.
- **COM-003** When the committer applies a change, the committer shall publish an `index.change.committed` event whose `refs` include the proposal and every verification event it counted.
- **COM-004** The committer shall apply commits in stream offset order.
- **COM-005** If a proposal's base commit differs from the target concept's current commit, then the committer shall publish an `index.change.conflicted` event and leave the index unchanged.
- **COM-006** If a human approver rejects a proposal, then the committer shall publish an `index.change.rejected` event that references the rejection.
- **COM-007** If a proposal's rule is not satisfied within the rule's expiry period, then the committer shall publish an `index.change.expired` event listing the missing verifications. **(OQ-6)**
- **COM-008** If the proposed content fails the OKF checks in §7.7, then the committer shall publish an `index.change.rejected` event listing each failure.
- **COM-009** If the write to the primary store fails, then the committer shall withhold the `index.change.committed` event until the write succeeds.
- **COM-010** When a rebuild is requested, the committer shall reproduce the index by replaying every `index.change.committed` event from the first offset.

### 7.7 Index content (IDX)

- **IDX-001** The index shall conform to OKF v0.2 (§11 of the OKF spec).
- **IDX-002** The index shall hold one concept per entity or thread, such as a customer, feature, decision, incident or policy. **(OQ-4)**
- **IDX-003** The committer shall accept a concept only when its frontmatter contains `type`, `title`, `description` and `generated` with `by` and `at`.
- **IDX-004** The committer shall set `generated.by` to the actor of the curator agent whose proposal was committed.
- **IDX-005** The committer shall add one `verified` entry for each vote, approval and check it counted, using that event's actor and time.
- **IDX-006** When a commit changes a concept's content, the committer shall keep the concept's existing `verified` entries with their original times.
- **IDX-007** The committer shall record each cited event as a `sources` entry with `id: evt-<offset>`, `resource` set to the event address, `author` set to the event actor, and `last_modified` set to the event time.
- **IDX-008** If a footnote label in a concept body matches no `sources[].id`, then the committer shall reject the proposal.
- **IDX-009** The committer shall set the extension key `commit` to the address of the concept's latest `index.change.committed` event. **(OQ-5)**
- **IDX-010** Where a concept was proposed because of earlier committed concepts or events, the committer shall list their addresses in the extension key `caused_by`. **(OQ-5)**
- **IDX-011** When the committer applies a change, the committer shall update the `index.md` of the concept's directory with the concept's link and description.
- **IDX-012** When the committer applies a change, the committer shall add a newest-first `log.md` entry naming the concept and the commit offset.
- **IDX-013** When a commit deprecates a concept, the committer shall set `status: deprecated` and keep the file.

### 7.8 Freshness (FRS)

- **FRS-001** When a raw event references the `resource` or a source of a committed concept, the committer shall ask the judgment service whether the event changes or invalidates a claim in that concept (JDG-050).
- **FRS-002** When the judgment service answers FRS-001 with a probability at or above the staleness threshold, the committer shall publish an `index.concept.stale` event that references the raw event and the concept's commit.
- **FRS-003** While an `index.concept.stale` event newer than a concept's latest commit exists, the context node shall report the concept as stale.
- **FRS-004** While the current time is on or after a concept's `stale_after`, the context node shall report the concept as stale.
- **FRS-005** While a concept is stale, the context node shall serve the last committed version to worker agents, together with a stale flag.

### 7.9 Worker agents (WRK)

- **WRK-001** The worker agent shall take organizational context only from the index and from `index.change.committed` events.
- **WRK-002** When an `index.change.committed` event reaches a context node, the context node shall refresh its cached copy of the changed concept.
- **WRK-003** When a worker agent uses index content in its output, the worker agent shall cite the concept path and its `commit` address.
- **WRK-004** When a worker agent follows a footnote to an event its role scope permits, the context node shall return the event.
- **WRK-005** If a worker agent follows a footnote to an event its role scope does not permit, then the context node shall return only the event's address and type.
- **WRK-006** When a worker agent changes a source system, the source system's connector shall publish the change to the stream as a raw event.

### 7.10 Index stores (STO)

- **STO-001** The index publisher shall write the index to exactly one primary store configured for the deployment. **(OQ-2)**
- **STO-002** Where the primary store or a mirror is a Git host (GitHub, GitLab), the index publisher shall create one git commit per `index.change.committed` event, with a message that contains the concept path and the event address.
- **STO-003** Where the primary store or a mirror is Confluence, the index publisher shall write each concept as one page and record the event address in each page version's comment.
- **STO-004** Where the primary store or a mirror is Google Drive, the index publisher shall write each concept as one file and record the event address in the file's properties.
- **STO-005** Where the primary store or a mirror is S3 or another object store, the index publisher shall write each concept as a versioned object with the event address in its metadata.
- **STO-006** Where mirrors are configured, when the primary write succeeds, the index publisher shall copy the change to every mirror.
- **STO-007** When the index is read back from any store, the index publisher shall return frontmatter and body text identical to the committed content.
- **STO-008** If a mirror's copy of a concept was changed outside the index publisher, then the index publisher shall publish the change as an `index.change.proposed` event attributed to the person who edited it. **(OQ-2)**
- **STO-009** Where a store has its own permissions, the index publisher shall grant read access to each concept only to users whose role scopes permit every event the concept cites. **(OQ-1)**

### 7.11 Traceability (TRC)

- **TRC-001** When a user or agent asks why a concept says what it says, the context node shall return every event reachable from the concept's `commit` through `refs`, in offset order.
- **TRC-002** If a returned chain includes an event the requester's role scope does not permit, then the context node shall show that event's address and type and mark it as withheld.
- **TRC-003** When a chain has no path from a concept back to a raw event, the context node shall report the offset at which the chain stops.
- **TRC-004** When a user asks why an event's outcome depended on a Jev answer, the context node shall show the judgment record from that event's payload.

### 7.12 Judgment services with Jev (JDG)

Jev supplies narrow, calibrated judgments; code applies the policy to them. Jev never writes index content, never makes the commit decision and never widens access.

**General**

- **JDG-001** The judgment service shall call Jev only through OpenRouter, with the pinned model ID `typesafe/jev-1.13`.
- **JDG-002** The judgment service shall read the OpenRouter credential from the environment of the host that runs it.
- **JDG-003** The judgment service shall return, for each question, the typed answer, the probability of each option or level, and the confidence.
- **JDG-004** The judgment service shall send all independent questions about the same state in one request.
- **JDG-005** When a Jev answer determines the outcome of an event, the judgment service shall add a judgment record to that event's payload.
- **JDG-006** If the state and questions for one request exceed 32,000 tokens, then the judgment service shall first select the relevant lines of the state with a Choice question over line IDs.
- **JDG-007** If a Jev request fails, then the judgment service shall return an explicit failure instead of a default answer.
- **JDG-008** Where the deployment limits which source scopes may be sent to OpenRouter, the judgment service shall send Jev only events from permitted scopes. **(OQ-11)**
- **JDG-009** The judgment service's callers shall take every threshold from the verification policy (POL-007).
- **JDG-010** The curator agent shall write proposed content with a generative model.
- **JDG-011** The committer shall make each commit decision in code, from the verification policy and the recorded judgments.

**Triage for curator agents**

- **JDG-020** When a raw event reaches a curator agent, the curator agent shall ask the judgment service, in one request: a Noul on whether the event holds a durable fact, decision or commitment; a Choice of kind of change from the verification policy, with a "none" option; and a Choice of the concept it concerns.
- **JDG-021** The curator agent shall supply the concept options from an index search, together with a "new entity" option.
- **JDG-022** If the durable-information probability is below the triage threshold, then the curator agent shall skip the event without calling a generative model.
- **JDG-023** When the chosen concept's confidence is at or above the concept-match threshold, the curator agent shall propose an update to that concept.
- **JDG-024** While the concept-match confidence is below the threshold, the curator agent shall give the generative model the three most probable concepts to decide between.

**Claim check**

- **JDG-030** When a proposal is published, the judgment service shall check each footnoted claim in the proposed content against the event its footnote cites.
- **JDG-031** When a claim quotes its source, the judgment service shall check in code that the quoted text occurs verbatim in the cited event before calling Jev.
- **JDG-032** If a quoted text does not occur in the cited event, then the judgment service shall mark the claim `fabricated`.
- **JDG-033** The judgment service shall classify each remaining claim with one Choice question whose options are `supports`, `contradicts` and `says nothing`.
- **JDG-034** When every claim is `supports` with confidence at or above the claim-check threshold, the committer shall publish a `verification.checked` event with the result `pass`.
- **JDG-035** If any claim is `fabricated` or `contradicts`, then the committer shall publish a `verification.checked` event with the result `fail` and escalate the proposal as in VER-011.
- **JDG-036** If any claim is `says nothing`, or has confidence below the claim-check threshold, then the committer shall route the proposal to a human approver.
- **JDG-037** The committer shall record each claim check with the actor `jev-claim-check/typesafe-jev-1.13`.

**Policy routing**

- **JDG-040** When a proposal is published, the committer shall ask the judgment service for the probability of each kind of change in the verification policy.
- **JDG-041** The committer shall apply the strictest rule whose probability is at or above the routing threshold, even when the curator agent classified the proposal as a less strict kind.
- **JDG-042** If the judgment service fails to classify a proposal, then the committer shall apply the rule for practice, policy or access changes.

**Staleness**

- **JDG-050** When the committer asks whether a raw event changes a concept (FRS-001), the judgment service shall answer with one Noul whose state holds the raw event and the concept's claims.

**Sensitivity**

- **JDG-060** When a raw event is appended, the stream shall ask the judgment service for the probability that the event contains each sensitive category the verification policy lists, such as personal data, HR, legal and security.
- **JDG-061** When a category's probability is at or above the sensitivity threshold, the stream shall publish an `event.scope.restricted` event naming the category.
- **JDG-062** The stream shall use `event.scope.restricted` events only to remove roles from the set that may read an event.
- **JDG-063** While an event's sensitivity judgment is pending or has failed, the stream shall withhold the event from every role that a restriction could remove.

**Worker agents**

- **JDG-070** Where index ranking is enabled, when a worker agent searches the index, the context node shall rank the candidate concepts with one Score question per candidate against the worker's task.
- **JDG-071** Where the output guardrail is enabled, when a worker agent produces output that cites the index, the context node shall ask the judgment service whether the output relies on claims absent from the cited concepts.
- **JDG-072** Where the output guardrail is enabled, if that probability is at or above the guardrail threshold, then the context node shall hold the output for its user's review.

## 8. Default verification policy

The first `policies/verification.md` committed to the index. The owner approves it, and changes to it follow POL-005.

| Kind of change | Verification before commit | Trust tier (OKF §5.3) | Expiry |
|----------------|----------------------------|-----------------------|--------|
| Customer fact from a call or email | Claim check passes, and two agents acting for different users agree; the proposer counts | machine-confirmed | 7 days |
| Interpretation of a system event | Claim check passes, and one agent, not the proposer's user, agrees | machine-confirmed | 3 days |
| Status proven by a system event | Deterministic check, no model | machine-confirmed | 1 day |
| New feature, commitment or decision | Claim check passes, agents agree, then the DRI approves | human-reviewed | 14 days |
| Practice, policy or access change | Claim check passes, then a named person approves; never automatic | human-reviewed | 14 days |

Thresholds used by judgment-service callers (POL-007):

| Threshold | Starting value | Used by |
|-----------|----------------|---------|
| Triage | 0.3 probability of durable information | JDG-022 |
| Concept match | 0.8 confidence | JDG-023, JDG-024 |
| Claim check | 0.8 confidence | JDG-034, JDG-036 |
| Routing | 0.2 probability for a stricter rule | JDG-041 |
| Staleness | 0.5 probability | FRS-002 |
| Sensitivity | 0.2 probability | JDG-061 |
| Guardrail | 0.5 probability | JDG-072 |

The expiry values and all thresholds are placeholders to be set by evaluation on the organization's own data **(OQ-6, OQ-10)**. The 0.8 claim-check value follows the TypeSafe citation-check recipe. The low routing and sensitivity values are deliberate, so that uncertainty leads to stricter handling.

## 9. Acceptance scenario

One customer call becomes one shipped feature. The offsets match the explainer page. Revision 2 adds a claim check before each vote and a triage judgment on each raw event; those events are not numbered here.

| # | Events | Expected outcome | Requirements |
|---|--------|------------------|--------------|
| 1 | 000184 `zoom.transcript.created` | Sensitivity check finds no restricted category. Delivered to sales-scoped agents only. Triage on Alex's node finds durable information about a new entity. Index unchanged. The worker agent receives nothing. | STR-004, ACC-002, ACC-004, JDG-020, JDG-060 |
| 2 | 000185 `index.change.proposed` by Alex's curator | Proposal cites 000184. Routing selects the customer-fact rule. Claim check passes. Index unchanged. | CUR-001, CUR-002, CUR-003, JDG-030, JDG-034, JDG-041 |
| 3 | 000186 `verification.vote` by Sam's agent; 000187 `index.change.committed` | `customers/acme.md` committed with `verified` from Sam's agent, tier machine-confirmed. The worker agent receives 000187. | VER-003, VER-005, COM-002, COM-003, IDX-005, WRK-002 |
| 4 | 000191 proposal for a feature; 000192 vote by Priya's agent | Agent votes satisfied; `verification.requested` to Dana. Index unchanged. | VER-006, VER-007 |
| 5 | 000203 `verification.approved` by `human:dana`; 000204 committed | Feature concept committed, tier human-reviewed, `caused_by` 000187. | VER-008, VER-009, IDX-005, IDX-010 |
| 6 | 000377 `github.pr.merged` | Jev judges that the event changes a claim in the feature concept; `index.concept.stale` published. Worker agents keep the last version with a stale flag. | FRS-001, FRS-002, FRS-003, FRS-005, JDG-050 |
| 7 | 000378 proposal, 000379 vote by Marco's agent, 000380 committed | Feature updated. Dana's `verified` entry keeps its 2026-04-29 time. | CUR-008, VER-005, IDX-006 |
| 8 | 000402 `release.published`, 000403 proposal, 000404 committed after a release check | `verification.checked` by `process:release-check`. "Why" on the feature returns 000404 back to 000184. | VER-010, TRC-001 |

## 10. Open questions

| ID | Question | Current leaning | Affects |
|----|----------|-----------------|---------|
| OQ-1 | Is the index one shared bundle, or does each reader see only concepts whose evidence they could read directly? | One index; each context node and store filters concepts by the reader's role scope. | STO-009, WRK-005 |
| OQ-2 | With several stores, which copy wins after a hand edit? | One primary store owned by the committer; mirrors are read-only and hand edits become proposals. | STO-001, STO-008 |
| OQ-3 | What happens when agents disagree? | Escalate to one human (the DRI), as in the Context Mesh design. | VER-011 |
| OQ-4 | What is one concept? | One per entity or thread; events are sources, not concepts. | IDX-002 |
| OQ-5 | What should the extension keys be called? `commit` and `caused_by`, or a single `lineage` map? | `commit` and `caused_by`. | IDX-009, IDX-010 |
| OQ-6 | What are the expiry periods, and does an expired proposal notify anyone? | Values in §8; notify the proposer's user. | COM-007, §8 |
| OQ-7 | On a Git primary store, should a proposal be a pull request, with votes as reviews? | Possibly for Git-only deployments. The stream stays the record, so the pull request would mirror the proposal. | CUR-001, STO-002 |
| OQ-8 | Can a worker agent ever act on an unverified proposal, for example during an incident? | No for v1; add an optional "provisional" feature later with its own `Where` requirements. | WRK-001 |
| OQ-9 | Are votes from different users' agents independent when they run the same model on the same evidence? | No. Count them as one vote; independent agreement needs different models or a person. | VER-013 |
| OQ-10 | What are the right thresholds? | Measure on labeled cases before enabling automatic commits. Start with the claim check on this repository's `chatgpt/` concepts against their transcripts. | POL-007, §8 |
| OQ-11 | Which source scopes may be sent to Jev through OpenRouter, given its data policy? | Decide per scope. Private scopes stay out until the owner approves, and their proposals go to a human. | JDG-008 |
| OQ-12 | What is the cost budget per event? | Measure `usage.cost` from judgment records during the pilot, then set a budget per source scope. | JDG-005 |
