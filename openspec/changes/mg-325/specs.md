# Feature Specification: Must-Gather Clean Integration with Must-Gather Operator

**Feature Branch**: `mg-325-must-gather-clean-integration`

**Created**: 2026-07-22

**Status**: Draft

**Input**: User description: "Based on Approach B, create an enhancement proposal for integration of must-gather-clean with must-gather-operator (MG-325)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Automated sanitization before case upload (Priority: P1)

A cluster administrator triggers a must-gather collection through the operator with sanitization enabled. The operator completes diagnostic collection, applies sanitization rules to remove or obfuscate sensitive data, and uploads only the sanitized bundle to the support case — without manual download-and-clean steps.

**Why this priority**: This is the core value of the integration — eliminating manual, error-prone post-processing while protecting confidential information before it leaves the cluster.

**Independent Test**: Can be fully tested by creating a must-gather request with sanitization enabled, verifying the uploaded bundle contains obfuscated sensitive fields and omits configured file patterns, while still preserving enough diagnostic structure for support analysis.

**Acceptance Scenarios**:

1. **Given** a must-gather request with sanitization enabled and valid case upload credentials, **When** collection completes successfully, **Then** the operator applies sanitization to the gathered output before upload and reports successful completion with sanitization applied.
2. **Given** a must-gather request with sanitization enabled, **When** the gather phase produces output containing known sensitive patterns (IP addresses, domain names, credentials), **Then** the uploaded bundle reflects obfuscated or omitted values per the active sanitization policy.
3. **Given** a must-gather request with sanitization enabled, **When** sanitization completes successfully, **Then** the administrator can observe that upload proceeded using the sanitized output without requiring manual intervention on a workstation.

---

### User Story 2 - Configurable sanitization policy per request (Priority: P2)

A cluster administrator or support engineer specifies which sanitization rules apply to a given must-gather run — including obfuscation types, domain allowlists/denylists, and file omission patterns — so that different cases or compliance requirements can be met without changing cluster-wide defaults.

**Why this priority**: Different support cases and customer environments require different levels of redaction; configurability prevents one-size-fits-all failures.

**Independent Test**: Can be tested by submitting two must-gather requests with different sanitization policies and confirming each uploaded bundle reflects its respective policy.

**Acceptance Scenarios**:

1. **Given** a must-gather request referencing a sanitization policy with domain-name obfuscation enabled, **When** the run completes, **Then** domain names in the uploaded bundle are consistently replaced per policy while non-sensitive metadata remains intact.
2. **Given** a must-gather request referencing a policy that omits specific file patterns, **When** sanitization runs, **Then** omitted files are absent from the uploaded bundle and a sanitization report indicates what was omitted.
3. **Given** a must-gather request with no sanitization policy specified, **When** sanitization is enabled at the request level, **Then** the system applies a documented default policy suitable for standard OpenShift support cases.

---

### User Story 3 - Sanitization failure handling and observability (Priority: P2)

A support engineer monitoring a must-gather run needs clear status when sanitization fails, partially completes, or blocks upload — so they can decide whether to retry, adjust policy, or proceed with a manual workflow.

**Why this priority**: Silent failures or uploading unsanitized data would undermine trust and compliance goals of the integration.

**Independent Test**: Can be tested by inducing a sanitization failure (invalid policy, unreadable gather output) and verifying status conditions and upload behavior match documented failure modes.

**Acceptance Scenarios**:

1. **Given** a must-gather request with sanitization enabled, **When** sanitization fails due to invalid policy configuration, **Then** the request reports a failed state with a reason indicating policy validation failure and does not upload an unsanitized bundle by default.
2. **Given** a must-gather request with sanitization enabled, **When** gather output is missing or incomplete, **Then** the request reports failure before upload and surfaces which phase (gather vs sanitization) did not succeed.
3. **Given** a completed must-gather run with sanitization applied, **When** an administrator inspects run status, **Then** they can determine whether sanitization ran, which policy version was applied, and whether upload used sanitized output.

---

### User Story 4 - Enhancement proposal review and approval (Priority: P1)

An operator maintainer or architect produces and socializes an enhancement proposal documenting how must-gather-clean integrates with the must-gather-operator under Approach B, so stakeholders can review architecture, scope, and verification strategy before implementation begins.

**Why this priority**: MG-325 explicitly requires a reviewed and approved proposal as the ticket deliverable; implementation must not proceed without alignment.

**Independent Test**: Can be tested by verifying the proposal document exists, covers required sections, and receives recorded approval from designated reviewers.

**Acceptance Scenarios**:

1. **Given** Approach B as the selected integration pattern, **When** the enhancement proposal is drafted, **Then** it documents end-to-end data flow from gather completion through sanitization to case upload, ownership boundaries between components, and operator-visible configuration surfaces.
2. **Given** a draft enhancement proposal, **When** reviewers evaluate it against MG-325 acceptance criteria, **Then** the proposal includes scope boundaries, migration/rollout considerations, verification strategy, and explicit out-of-scope items.
3. **Given** an approved enhancement proposal, **When** downstream planning begins, **Then** implementers can derive functional requirements and test plans without re-interpreting the original Jira ticket or external reference document alone.

---

### Edge Cases

- **When** sanitization is enabled but gather fails, **then** no upload occurs and status reflects gather failure without attempting sanitization on empty or partial output.
- **When** sanitization is enabled and policy validation fails at request admission time, **then** the request is rejected before any gather workload starts, with a clear validation message.
- **When** sanitization succeeds but upload credentials are invalid, **then** sanitized output remains available for retry according to existing operator retention rules and status indicates upload failure distinct from sanitization failure.
- **When** a must-gather request disables sanitization explicitly, **then** behavior matches today's operator flow (gather and upload without an automated sanitization step).
- **When** sanitization produces a report of omitted or obfuscated items, **then** the report is retained for the lifetime of the must-gather run resources or included in uploaded metadata per policy [NEEDS CLARIFICATION: whether sanitization reports are uploaded to the case, stored only on-cluster, or both].
- **When** an cluster upgrades while must-gather runs are in progress, **then** in-flight runs complete or fail predictably without uploading partially sanitized bundles.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST support an optional sanitization step in the must-gather operator workflow that processes gathered diagnostic output before case upload when enabled on a request.
- **FR-002**: System MUST apply must-gather-clean-equivalent obfuscation and omission rules to gathered output, including support for configurable policies that define obfuscator types, domain lists, keyword patterns, and file omission rules.
- **FR-003**: System MUST upload only sanitized output to the support case when sanitization is enabled and completes successfully, unless an explicit override policy is defined [NEEDS CLARIFICATION: whether upload-on-partial-sanitization is ever permitted].
- **FR-004**: Administrator MUST be able to enable or disable sanitization per must-gather request without affecting unrelated operator functions or other concurrent requests.
- **FR-005**: System MUST reject invalid sanitization policy configuration before starting gather workloads, with operator-visible validation errors.
- **FR-006**: System MUST surface distinct status for gather, sanitization, and upload phases so operators can identify which step failed.
- **FR-007**: System MUST produce a sanitization summary indicating policy applied, counts or categories of obfuscated/omitted content, and completion outcome.
- **FR-008**: System MUST preserve backward compatibility — requests without sanitization enabled MUST behave as they do prior to this integration.
- **FR-009**: Enhancement proposal MUST document Approach B integration architecture, configuration model, failure modes, scope boundaries, verification strategy, and rollout/migration considerations for reviewer approval.
- **FR-010**: Enhancement proposal MUST identify impacted components (must-gather operator, must-gather-clean tooling, case upload path) and data-handling boundaries without requiring readers to access external-only documents.

### Key Entities

- **Must-Gather Request**: A user-initiated diagnostic collection request with optional sanitization enablement, policy reference, upload target, and lifecycle status across gather, sanitization, and upload phases.
- **Sanitization Policy**: A declarative ruleset defining obfuscators (IP, MAC, domain, keyword, regex), omission patterns, and domain-specific allow/deny lists applied to gather output.
- **Gather Output**: The raw diagnostic bundle produced by the gather phase, stored in shared storage accessible to subsequent pipeline steps.
- **Sanitized Output**: The post-processed bundle ready for upload, derived from gather output per active policy, with consistent replacement values to preserve debuggability where configured.
- **Sanitization Report**: A summary artifact describing policy version, obfuscation/omission actions taken, errors encountered, and timestamps.
- **Enhancement Proposal**: The MG-325 deliverable documenting Approach B integration design, acceptance mapping, and reviewer sign-off criteria.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A cluster administrator can enable sanitization on a must-gather request and receive a completed, uploaded case attachment without performing manual must-gather-clean steps on a workstation.
- **SC-002**: In test scenarios with seeded sensitive values (IP addresses, domain names, credentials), 100% of configured sensitive patterns in the uploaded bundle are obfuscated or omitted per the active policy.
- **SC-003**: Invalid sanitization policies are rejected before gather starts in 100% of test cases, with operator-visible error messages identifying the validation failure.
- **SC-004**: When sanitization fails after successful gather, zero unsanitized bundles are uploaded to the case by default across test scenarios.
- **SC-005**: Sanitization status (pending, running, succeeded, failed) is observable on the must-gather request within one reconciliation cycle of phase transition.
- **SC-006**: The enhancement proposal receives documented approval from designated reviewers and covers all sections required by FR-009 and FR-010 before implementation planning begins.
- **SC-007**: Requests with sanitization disabled complete with the same observable outcomes as pre-integration behavior in regression test scenarios.

## Assumptions

- **A-001**: Primary actors are cluster administrators (initiate requests), support engineers (monitor outcomes), and operator maintainers (author/review the enhancement proposal).
- **A-002**: **Approach B** integrates sanitization as an automated in-cluster step in the operator job pipeline between gather completion and case upload, reusing shared storage between pipeline phases rather than requiring manual off-cluster cleaning (per referenced design document; full Approach B details to be inlined in the enhancement proposal).
- **A-003**: **must-gather-clean** provides the sanitization engine (obfuscation, omission, reporting); the operator orchestrates when and how it runs but does not reimplement sanitization logic.
- **A-004**: **must-gather-clean-test** referenced in the ticket title is a validation/test harness for must-gather-clean behavior; the integration target is the production must-gather-clean tooling paired with must-gather-operator.
- **A-005**: Sanitization is **opt-in per request** by default to preserve backward compatibility and allow cases where full fidelity is intentionally required.
- **A-006**: A default OpenShift-oriented sanitization policy (equivalent to must-gather-clean's standard example policy) is available when no custom policy is specified.
- **A-007**: MVP scope covers obfuscation of IP addresses, MAC addresses, domain names, and keyword/regex patterns plus configurable file omission; advanced custom obfuscator plugins are out of scope unless added during proposal review.
- **A-008**: Case upload via existing SFTP/case-management path remains unchanged except that the input to compression/upload is sanitized output when sanitization is enabled.
- **A-009**: Hypershift-specific, disconnected/air-gapped mirror, and non-OpenShift Kubernetes distributions are out of scope unless explicitly added during proposal review.
- **A-010**: MG-325 delivery is proposal-first; implementation of FR-001 through FR-008 follows separate planning after proposal approval (per ticket acceptance criterion).
- **A-011**: Context and motivation — support cases often contain sensitive customer data; manual must-gather-clean usage creates friction and inconsistency; automating sanitization in the operator reduces exposure risk and operator toil.
- **A-012**: Dependencies — must-gather-clean container image availability, compatible policy schema, and sufficient shared storage for gather and sanitized outputs during the job lifecycle.
