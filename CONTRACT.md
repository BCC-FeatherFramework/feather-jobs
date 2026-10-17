# Feather Jobs: Contract 1

**Status:** proposed for freezing with feather-jobs 0.2.0 (2026-10-17).

This document is the public contract of feather-jobs: what other resources may
call and rely on. Once frozen, everything listed here keeps working, with the
same meaning, for as long as `Contract = 1`. The README explains the model and
setup; this file is the reference a consumer codes against.

## 1. Compatibility promise

Within Contract 1:

- **May be added** without a new contract version: optional request fields,
  result fields, exports, events, payload fields, error codes, capability
  features and employer types. Consumers must ignore unknown result fields and
  treat an unknown error code as a failure.
- **Never changes**: the name, required fields, result shape or meaning of
  anything below; an error code's meaning; an event's name or the meaning of
  its fields; the policy action names; the duty states; idempotency and
  revision rules.
- **Needs Contract 2**: removing or renaming anything, making an optional field
  required, or changing what a result or event means.

Consumers check `GetCapabilities().value.contract == 1` and the features they
use before relying on them.

## 2. Lifecycle

| Export | Result |
|---|---|
| `GetCapabilities()` | `{ resource, contract = 1, version, state, features }` |
| `GetHealth()` | `{ state, phase, ... }` (diagnostic; fields may be added) |
| `AwaitReady(timeoutMs)` | The health snapshot once ready; `not_ready` after the timeout, `startup_failed` if startup failed. `timeoutMs` 0 to 60000, default 30000 |

Features in Contract 1 (value `1`): `lifecycle`, `health`, `migrations`,
`definitions`, `grades`, `assignments`, `multipleAssignments`, `duty`,
`revisions`, `idempotency`, `operationLookup`, `outbox`, `eventPublication`,
`auditHistory`, `employerValidation`, `clientSnapshot`, `references`.
`activeAssignment` is `1` when the server enables active selection
(`Config.ActiveAssignment.enabled`, shipped on), else `0`. `societyAdapter` and
`legacyMigration` are `0`: society hiring lives in the consumer (bcc-society),
and Jobs imports no legacy job data.

## 3. Calling rules

- **Results.** Every export returns `{ ok = true, value = ... }` or
  `{ ok = false, code, message, details }`. `message` is for logs; branch on
  `code`.
- **Caller.** The calling resource is taken from Cfx (`GetInvokingResource`);
  it must be in the matching `Config.Access` group. No request field can claim
  another resource, account or character.
- **Common mutation fields.** `requestId` (required, 1 to 128 characters,
  `^[A-Za-z0-9][A-Za-z0-9._:-]*$`), `reasonCode` (required, namespaced,
  `^[a-z][a-z0-9._:-]*$`, at most 64), optional `actorSource` (the connected
  player who triggered it; Jobs derives the account and character from Core),
  optional `correlationId` (same pattern as `requestId`), optional
  `referenceType` (`^[a-z][a-z0-9_]*$`, at most 48) + `referenceId` (same
  pattern as `requestId`), both or neither: the caller's record behind the
  change. Unknown fields are rejected.
- **Idempotency.** A request ID is scoped to the calling resource. Repeating it
  with the same payload returns the stored result with `replayed = true`;
  reusing it with a different payload returns `idempotency_conflict`. A replay
  returns the result as it was stored when the request first succeeded. After a
  timeout, retry with the same request ID or call `FindOperation`.
- **Revisions.** Changes to an existing definition, grade or assignment take
  its `expectedRevision`; `SetActiveAssignment` takes the character state
  revision (`0` before any selection). A stale value returns
  `revision_conflict` with `details.revision`.
- **Ownership.** The resource that registers a definition owns it and its
  grades; the resource that creates an assignment owns it. Only the owner, or a
  privileged registrar or assigner, may change them. A resource may only assign
  jobs it owns unless `Config.Assignments.allowForeignJobs` is on or it is a
  privileged assigner.
- **Policy.** Mutations are also checked through Core policy, with the caller as
  `subject.resource`:
  - `jobs.definition.manage`: register and update definitions, define grades;
  - `jobs.assignment.manage`: create, change grade, duty, active selection, end.

  A missing or failing policy provider denies (`policy_unavailable`).

## 4. Definitions and grades

Callers must be in `trustedRegistrars`.

| Export | Fields (besides the common ones) | Result |
|---|---|---|
| `RegisterJobDefinition` | `jobKey`, `label`, optional `metadataVersion` | Definition snapshot |
| `UpdateJobDefinition` | `jobId`, `expectedRevision`, at least one of `label`, `status` (`active`, `retired`), `metadataVersion` | Definition snapshot |
| `DefineGrade` | `jobId`, `gradeKey`, `label`, `ordinal`, optional `status`, `metadataVersion`; `expectedRevision` to update an existing grade | Grade snapshot |

Field formats: `jobKey` and `gradeKey` are `^[a-z][a-z0-9_]*$`, at most 48;
`label` is 1 to 100 characters without control characters; `ordinal` is an
integer 0 to 1000; `metadataVersion` is an integer 1 to 65535 (a version number
for the consumer; Jobs stores no metadata body).

Rules that are part of the contract:

- A job key is unique across the server. Registering a taken key returns
  `job_key_conflict` with `details.jobId` and `details.ownerResource`.
  Consumers re-register on every start with the same request ID, which replays.
- Ordinals are unique per job and order the grades (higher is more senior).
  `grade_ordinal_conflict` names the grade holding the ordinal; move a grade to
  a free ordinal before swapping. At most `Config.Limits.maxGradesPerJob`
  grades per job (`grade_limit`).
- `DefineGrade` without `expectedRevision` creates the grade (`grade_exists`
  with `details.gradeId` and `details.revision` if it exists); with it, it
  updates the grade (`grade_not_found` if it does not exist).
- A retired job or grade accepts no new assignments or promotions
  (`job_inactive`, `grade_inactive`). Assignments that already hold it stay
  open; snapshots report `jobStatus` so the consumer can decide.

## 5. Assignments

Callers must be in `trustedAssigners`.

| Export | Fields (besides the common ones) | Result |
|---|---|---|
| `CreateAssignment` | `characterId`, `jobId` or `jobKey`, `gradeId` or `gradeKey`, `provenanceType`, `provenanceId`, optional `employerType` + `employerId`, `dutyState` (default `off_duty`) | Assignment snapshot, plus `active = true` when it became the active assignment |
| `ChangeAssignmentGrade` | `assignmentId`, `expectedRevision`, `gradeId` or `gradeKey` (same job) | Assignment snapshot |
| `SetDutyState` | `assignmentId`, `expectedRevision`, `dutyState` | Assignment snapshot |
| `SetActiveAssignment` | `characterId`, `assignmentId` (or `false` to clear), `expectedRevision` (character state) | `{ characterId, activeAssignmentId, previousActiveAssignmentId, revision }` |
| `EndAssignment` | `assignmentId`, `expectedRevision` | Assignment snapshot |

Field formats: `characterId`, `assignmentId`, `jobId`, `gradeId`, `employerId`
are UUIDs (compared in lower case); `employerType` is a configured employer
type; `provenanceType` is `^[a-z][a-z0-9_]*$`, at most 48; `provenanceId`
follows the `requestId` pattern.

Rules that are part of the contract:

- The character must resolve through the `character-profile` provider and be
  active; offline characters are valid. `organization` employers must exist and
  be active in feather-organizations; `society` employers are trusted to the
  owning resource.
- A character holds at most one open assignment per job and employer
  (`assignment_exists` with `details.assignmentId`), so two employers never
  overwrite each other, and at most `Config.Assignments.maxOpenPerCharacter`
  open assignments (`assignment_limit`).
- **Duty** is one of `off_duty`, `on_duty`, `suspended`, per assignment. A duty
  change never changes the job or grade. Setting the current duty returns
  `no_change`.
- **Active assignment.** With active selection enabled, a character's first
  open assignment becomes active automatically. `SetActiveAssignment` selects
  one of the character's open assignments owned by the caller; replacing or
  clearing a selection that points at another resource's assignment needs a
  privileged assigner. When disabled it returns `feature_disabled`.
- `EndAssignment` is terminal: status `ended`, duty reset to `off_duty`, and an
  active selection pointing at it is cleared in the same commit. Later changes
  return `assignment_ended`.
- Changing to the grade the assignment already holds returns `no_change`.

## 6. Reads

Callers must be in `trustedReaders` (history: `trustedAuditors`).

| Export | Fields | Result |
|---|---|---|
| `GetJobDefinition` | `jobId` or `jobKey` | Definition snapshot with `grades` ordered by ordinal |
| `ListJobDefinitions` | optional `ownerResource`, `status`, `limit` (1 to 50, default 20), `cursor` | `{ items = { definition... }, nextCursor }`, ordered by job key; the cursor is a job key |
| `GetAssignment` | `assignmentId` | Assignment snapshot |
| `ListCharacterAssignments` | `characterId`, optional `includeEnded`, `limit` (1 to 50, default 20), `cursor` | `{ items = { assignment... }, nextCursor }`, newest first; the cursor is an assignment UUID |
| `GetEffectiveJobSnapshot` | `characterId` | `{ characterId, revision, activeAssignmentId, active, assignments }`: open assignments, oldest first |
| `CheckAssignment` | `characterId`, `jobKey`, optional `gradeKey`, `minGradeOrdinal`, `employerType`, `employerId`, `dutyState`, `activeOnly` | `{ matched, assignment? }`: the first open assignment matching every filter |
| `FindOperation` | `requestId` | `{ found = false }` or `{ found = true, operation, result }` for the caller's own request |
| `InspectJobHistory` | `aggregateType` (`definition`, `grade`, `assignment`, `character`), `aggregateId`, optional `limit` (1 to 50, default 20), `cursor` | `{ items = { event... }, nextCursor }`, newest first; an owner reads its own definitions, grades and assignments, a privileged auditor anything including a character's whole trail |

`CheckAssignment` returns facts only. The calling resource decides whether they
allow an action (for example, a door that requires duty passes
`dutyState = 'on_duty'`).

`FindOperation` operations: `definition.register`, `definition.update`,
`grade.define`, `assignment.create`, `assignment.grade`, `assignment.duty`,
`assignment.activate`, `assignment.end`.

### Snapshots

```lua
-- Definition
{ jobId, jobKey = 'blacksmith', label, ownerResource, status = 'active', -- active | retired
  metadataVersion, revision, grades = { grade... } }                     -- grades: GetJobDefinition only

-- Grade
{ gradeId, jobId, gradeKey = 'apprentice', label, ordinal = 0, status, metadataVersion, revision }

-- Assignment
{
    assignmentId, characterId,
    jobId, jobKey, jobLabel, jobStatus,
    gradeId, gradeKey, gradeLabel, gradeOrdinal,
    employerType = 'organization', employerId,   -- both nil without an employer
    status = 'active',                           -- active | ended
    dutyState = 'off_duty',                      -- off_duty | on_duty | suspended
    ownerResource, provenanceType, provenanceId,
    revision,
    assignedAt = 1759500000000,                  -- Unix ms (second precision)
    endedAt = nil
}
```

History items: `{ eventId, eventType, aggregateType, aggregateId, jobId,
characterId, sourceResource, requestId, reasonCode, referenceType, referenceId,
revision, actorAccountId, actorCharacterId, correlationId, details, createdAt }`,
where `eventType` is the short form (section 8) and `details` holds the event's
extra fields.

## 7. Error codes

| Code | Meaning |
|---|---|
| `invalid_input` | A field is missing, unknown or malformed |
| `not_ready` | Jobs is starting or stopped |
| `startup_failed` | `AwaitReady`: Jobs failed to start; `details.health` |
| `dependency_unavailable` | Core is unavailable |
| `authorization_denied` | Caller not in the access group, not the owner, foreign job, or policy denied |
| `policy_unavailable` | The policy provider is missing or failed (fails closed) |
| `actor_unavailable` | `actorSource` has no active character session |
| `idempotency_conflict` | Request ID reused with a different payload |
| `revision_conflict` | Stale `expectedRevision`; `details.revision` is current |
| `transaction_failed` | Commit not confirmed; retry the same request ID or call `FindOperation` |
| `job_key_conflict` | Job key already registered; `details.jobId`, `details.ownerResource` |
| `job_not_found`, `job_inactive` | No such job, or it is retired |
| `grade_not_found`, `grade_inactive` | No such grade on this job, or it is retired |
| `grade_exists` | `DefineGrade` without `expectedRevision` for an existing grade; `details.gradeId`, `details.revision` |
| `grade_ordinal_conflict` | Another grade uses the ordinal; `details.gradeKey` |
| `grade_limit` | The job has reached its grade limit |
| `assignment_exists` | Open assignment for this job and employer; `details.assignmentId` |
| `assignment_limit` | The character holds the maximum number of open assignments |
| `assignment_not_found`, `assignment_ended` | No such assignment (for this character), or it has ended |
| `no_change` | The request matches the current state |
| `feature_disabled` | Active selection is disabled on this server |
| `character_not_found`, `character_inactive`, `character_provider_unavailable` | Character validation |
| `employer_not_found`, `employer_inactive`, `employer_provider_unavailable` | Employer validation |
| `not_found` | History: no such aggregate |
| `invalid_cursor` | The cursor is not from this listing |
| `character_required` | Client route: no active character |
| `invalid_persistence`, `internal_error` | Stored data is invalid, or an unexpected failure (report it) |

## 8. Events

Published through Core after commit, from an outbox, so each committed fact is
published even across restarts. Delivery is at least once: consumers deduplicate
by `eventId`. Server-only.

| Event | When | Extra fields |
|---|---|---|
| `jobs.definition.registered.v1` | A definition is registered | `jobKey`, `label`, `ownerResource`, `status` |
| `jobs.definition.updated.v1` | A definition changes | `jobKey`, `label`, `status`, `previousStatus`, `metadataVersion` |
| `jobs.grade.defined.v1` | A grade is created or changed | `jobKey`, `gradeId`, `gradeKey`, `label`, `ordinal`, `status`, `previousStatus` |
| `jobs.assignment.created.v1` | An assignment is created | `jobKey`, `gradeId`, `gradeKey`, `dutyState`, `employerType`, `employerId`, `provenanceType`, `provenanceId`, `ownerResource` |
| `jobs.assignment.grade_changed.v1` | The grade changes | `jobKey`, `gradeId`, `gradeKey`, `previousGradeId`, `previousGradeKey` |
| `jobs.assignment.duty_changed.v1` | Duty changes | `jobKey`, `dutyState`, `previousDutyState` |
| `jobs.assignment.activated.v1` | The active selection changes (including automatic selection and clearing) | `jobKey`, `activeAssignmentId`, `previousActiveAssignmentId`, `activeCleared` |
| `jobs.assignment.ended.v1` | An assignment ends | `jobKey`, `status`, `previousStatus`, `previousDutyState`, `activeCleared` |

Every payload carries `eventId`, `aggregateType`, `aggregateId`, `jobId`,
`characterId`, `assignmentId`, `revision`, `sourceResource`, `requestId`,
`reasonCode`, `referenceType`, `referenceId`, `actorAccountId`,
`actorCharacterId` and `correlationId` (absent when not applicable). For
`jobs.assignment.activated.v1` the aggregate is the character and `revision` is
the character state revision. Payloads are at most 4096 bytes.

## 9. Client route

`jobs.snapshot.get.v1`, empty payload, returns
`{ activeAssignmentId, assignments = { { assignmentId, jobKey, jobLabel, gradeKey, gradeLabel, gradeOrdinal, employerType, employerId, dutyState } } }`:
the calling player's own open assignments, without owner or provenance. It
requires an active character and is rate limited (5 calls per 5 seconds, 64-byte
payload). Clients cannot change anything.

## 10. Configuration

A server may change:

- the access groups (`trustedReaders`, `trustedRegistrars`,
  `privilegedRegistrars`, `trustedAssigners`, `privilegedAssigners`,
  `trustedAuditors`, `privilegedAuditors`);
- `Config.Assignments` (open assignment cap, foreign jobs),
  `Config.ActiveAssignment`, `Config.Limits` and outbox timing;
- add employer types;
- `Config.Authorization` action names only together with matching policy grants.

A server must not, once data exists:

- rename or remove an employer type key (it is stored on assignments);
- lower `maxGradesPerJob` or `maxOpenPerCharacter` below what is already stored
  (existing rows stay, new ones are refused);
- edit an applied migration (a checksum mismatch fails startup).

Job and grade keys are data, not configuration: they are registered by their
owning resource and never renamed; retire a job or grade instead.

## 11. Not in Contract 1

Wages, payroll, licenses and qualifications, society ranks and their mapping to
grades, and importing legacy job data are not part of this contract. They stay
in the consumers or other domains; anything new would arrive as exports or
capability features announced through `GetCapabilities`.
