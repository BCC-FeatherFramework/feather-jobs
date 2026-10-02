# Feather Jobs

Authoritative job domain for the Feather Framework: job definitions, grades,
character assignments, the optional active assignment, and per-assignment duty
state. Independent of societies, ranks, wages, payroll, uniforms, locations,
licenses, and general permissions; those remain with gameplay resources and
other domains. Native Feather; no legacy compatibility layer.

Implements phases J0–J3 of the Jobs master plan (`feather-framework-docs`,
`feather-jobs/Feather_Jobs_Master_Plan.md`) on the Contract 1 service kernel. J4
(Society adapter) is delivered by the `bcc-society` rewrite. J5 (consumer
cutover) waits for the remaining job-reading scripts' Feather rewrites, and J6
(legacy removal) has nothing left on the Feather side.

Shipped defaults use `Config.DevMode=false` with service-policy authorization
enabled. Acceptance fixtures receive no trust and development commands are not
registered. The permanent server-console `JobsReleaseContractSmokeTest` is
read-only and checks readiness, policy mode, development-command and fixture
exclusion, publisher state, migrations, the client route, and capabilities.
Enable DevMode only on an isolated development server; disable it and restart
before packaging.

Production-surface acceptance passed 10/10 on 2026-10-02: Jobs ready, DevMode
disabled, development commands absent, fixture trust absent, authorization
enabled, publisher running, migration ledger intact, client route owned,
untrusted reader denied, and capabilities advertised. bcc-society started on
this configuration with its job types, organization link and owner assignment.

Development acceptance passed on 2026-10-02:

- `JobsLifecycleSmokeTest` passed 20/20 (registration, idempotent replay and
  conflict, untrusted denial, grade ordinals, open-slot uniqueness, stale
  revision, grade change, duty, foreign mutation denial, terminal end,
  operation lookup, audit history).
- `JobsConsumerLiveTest` (from `[test]/feather-jobs-tests`) passed 14/14 from a
  second resource: caller ownership, consumer grants, foreign-job denial, and
  event delivery.
- `SocietyLiveTest` (bcc-society) passed 9/9 against Jobs as the reference
  consumer.

## Dependencies

```text
feather-mysql
feather-core
```

At runtime, assignment creation also needs the `character-profile` provider
(feather-character). Organization employers are validated through
feather-organizations when that employer type is used. Service policy is decided
by the default Core policy provider (feather-admin).

## Startup order

Start Jobs after Core, Character, Organizations and Authority, and before Admin
and job consumers:

```text
ensure feather-mysql
ensure feather-core
ensure feather-character
ensure feather-organizations
ensure feather-authority
ensure feather-jobs
ensure feather-admin
```

The first start applies three checksummed migrations and logs
`event=startup.ready migrationsApplied=3 routes=1`. Later starts apply none.

## Installation and first acceptance

Place in `resources/[feather]/feather-jobs`, add it to `server.cfg` in the order
above, and run in the server console:

```text
refresh
ensure feather-jobs
JobsReleaseContractSmokeTest
```

Expect 10/10 with the shipped configuration.

The mutating acceptance tests need `Config.DevMode=true` on an isolated
development server. Enable it, restart feather-jobs, and run with a character
loaded:

```text
JobsLifecycleSmokeTest <serverId|characterUuid>
```

Expect 20/20. The test mutates data: it registers a uniquely keyed `smoke_*`
job, assigns it, walks the lifecycle, ends the assignment, and retires the job.

For the second-resource check, also add the fixture's policy grant
(`Config.servicePolicy['feather-jobs']['feather-jobs-tests']` with
`jobs.definition.manage` and `jobs.assignment.manage`) to feather-admin, restart
feather-admin, then:

```text
ensure feather-jobs-tests
JobsConsumerLiveTest <serverId>
```

Expect 14/14. The fixture is described in `[test]/feather-jobs-tests/README.md`.
Remove the fixture grant and set DevMode back to `false` afterwards.

If the first mutation returns `authorization_denied`, restart feather-admin so it
loads the `['feather-jobs']` service-policy grants.

## Production posture

The shipped configuration is the production posture. Run after startup:

```text
JobsReleaseContractSmokeTest
```

Expect 10/10: the development lifecycle command is not registered and the
`feather-jobs-tests` fixture receives no trust. Feather-admin's service policy
grants Jobs actions only to `feather-jobs`, `feather-admin` and `bcc-society`.

## Ownership and decisions

Jobs owns job definitions and stable keys, grade definitions and ordering,
character assignments with their lifecycle and provenance, explicit duty state,
the optional active assignment, and the audit facts for all of them.

It does not own society or business definitions, society ranks or manager
permissions, job locations, NPCs, prompts, uniforms, crafting, missions, wage
rules, payroll, licenses, or general authorization roles.

The J0 decisions follow the master plan's recommendations and are configurable:

- **Multiple assignments:** up to `Config.Assignments.maxOpenPerCharacter` (8)
  open assignments per character.
- **Active assignment:** optional (`Config.ActiveAssignment.enabled`). A
  character's first open assignment becomes active automatically.
- **Duty:** a per-assignment enum: `off_duty`, `on_duty`, `suspended`. Never an
  `off` prefix on a job name.
- **Definition governance:** definitions are persisted in Jobs and owned by the
  registering resource. A resource may only assign jobs it owns unless
  `Config.Assignments.allowForeignJobs` is enabled or it is privileged.
- **Society ranks:** an explicit rank → grade mapping, owned by the consumer.
  Jobs never infers it.
- **Offline changes:** allowed. The character UUID resolves through the Character
  provider.

An open assignment is unique per character + job + employer, so two employers
can assign the same job to the same character without overwriting each other.

## Server exports

All exports return the Contract 1 envelope `{ ok = true, value = ... }` or the
flat `{ ok = false, code, message, details }`. The caller is the actual Cfx
invoking resource and must be listed in the matching `Config.Access` group. No
client may supply a trusted resource, account, or character identity.

`GetCapabilities()`, `GetHealth()`, and `AwaitReady(timeoutMs)` (default 30000,
integer 0–60000) report lifecycle.

Every mutation takes a stable `requestId` (at most 128 characters) and a
namespaced `reasonCode` such as `society.hire`. It also accepts an optional
`actorSource` (the player who triggered it; Jobs derives that player's account
and character from Core) and an optional `correlationId`. Retrying a request ID
with the same payload replays the stored result with `replayed = true`; reusing it
with a different payload returns `idempotency_conflict`.

- `RegisterJobDefinition` — `jobKey`, `label`, optional `metadataVersion`. The caller becomes the owner.
- `UpdateJobDefinition` — `jobId`, `expectedRevision`, and any of `label`, `status`, `metadataVersion`.
- `DefineGrade` — `jobId`, `gradeKey`, `label`, `ordinal` (0–1000, unique per job), optional `status`. Pass `expectedRevision` to update an existing grade.
- `CreateAssignment` — `characterId`, `jobId` or `jobKey`, `gradeId` or `gradeKey`, `provenanceType`, `provenanceId`, optional `employerType` + `employerId`, optional `dutyState`.
- `ChangeAssignmentGrade` — `assignmentId`, `expectedRevision`, `gradeId` or `gradeKey` (same job).
- `SetDutyState` — `assignmentId`, `expectedRevision`, `dutyState`. Never changes job or grade.
- `SetActiveAssignment` — `characterId`, `assignmentId` (or `false` to clear), `expectedRevision` of the character state (`0` initially).
- `EndAssignment` — `assignmentId`, `expectedRevision`. Terminal; resets duty and clears the active selection when it pointed here.

Reads:

- `GetJobDefinition({ jobId })` or `({ jobKey })` — the definition with grades by ordinal.
- `ListJobDefinitions({ ownerResource?, status?, limit?, cursor? })` — bounded pages.
- `GetAssignment({ assignmentId })`
- `ListCharacterAssignments({ characterId, includeEnded?, limit?, cursor? })` — newest first.
- `GetEffectiveJobSnapshot({ characterId })` — active assignment, open assignments, state revision.
- `CheckAssignment({ characterId, jobKey, gradeKey?, minGradeOrdinal?, employerType?, employerId?, dutyState?, activeOnly? })` — returns `{ matched, assignment? }`.
- `FindOperation({ requestId })` — the caller's stored result, for timeout recovery.
- `InspectJobHistory({ aggregateType, aggregateId, limit?, cursor? })` — audit trail; owners read their own aggregates, privileged auditors read all.

`CheckAssignment` returns facts only. The consuming resource still decides
whether those facts permit an action.

Common failure codes: `authorization_denied`, `policy_unavailable` (Jobs fails
closed), `revision_conflict` (`details.revision` holds the current value),
`idempotency_conflict`, `assignment_exists` (`details.assignmentId`),
`job_key_conflict`, `no_change`, and `transaction_failed` (retry with the same
request ID, or call `FindOperation` first).

## Contract

```lua
local Jobs = exports['feather-jobs']

-- On start: register owned definitions with stable request IDs (replays on restart).
local job = Jobs:RegisterJobDefinition({
    requestId = 'definition:blacksmith', jobKey = 'blacksmith',
    label = 'Blacksmith', reasonCode = 'society.setup'
})
Jobs:DefineGrade({
    requestId = 'grade:blacksmith:apprentice', jobId = job.value.jobId,
    gradeKey = 'apprentice', label = 'Apprentice', ordinal = 0,
    reasonCode = 'society.setup'
})

-- Hire: the consumer has validated its own rule and stored a pending membership.
local result = Jobs:CreateAssignment({
    requestId = ('hire:%s'):format(membershipId),
    characterId = targetCharacterId,
    jobKey = 'blacksmith', gradeKey = 'apprentice',
    employerType = 'organization', employerId = organizationId,
    provenanceType = 'society_membership', provenanceId = membershipId,
    reasonCode = 'society.hire', actorSource = source
})

-- Elsewhere: facts for a door, crafting bench or billing rule.
local check = Jobs:CheckAssignment({
    characterId = characterId, jobKey = 'blacksmith',
    minGradeOrdinal = 10, dutyState = 'on_duty'
})
```

## Client route

`jobs.snapshot.get.v1` takes an empty payload and returns the calling player's
own open assignments, without owner or provenance internals. It requires an
active character and is rate limited. Clients cannot mutate anything.

## Event publication

Committed facts are written to the `feather_job_events` audit table and the
`feather_job_outbox` in the same transaction as the change, then published
through Core:

```text
jobs.definition.registered.v1
jobs.definition.updated.v1
jobs.grade.defined.v1
jobs.assignment.created.v1
jobs.assignment.grade_changed.v1
jobs.assignment.duty_changed.v1
jobs.assignment.activated.v1
jobs.assignment.ended.v1
```

`jobs.definition.updated.v1` and `jobs.grade.defined.v1` extend the master plan's
list so definition and grade changes carry the same audit trail as assignments.
Payloads are bounded to 4096 bytes.

## Configuration and access

`Config.Access` holds the trusted resource groups: `trustedReaders`,
`trustedRegistrars` (with `privilegedRegistrars`), `trustedAssigners` (with
`privilegedAssigners`), and `trustedAuditors` (with `privilegedAuditors`). A
privileged entry must also be in its trusted group. Each consumer additionally
needs `jobs.definition.manage` and/or `jobs.assignment.manage` in
`Config.servicePolicy['feather-jobs'][<resource>]` of feather-admin; restart
feather-admin after changing it.

`Config.EmployerTypes` lists the accepted employer types: `organization`
(validated through feather-organizations, which must list `feather-jobs` in its
`trustedReaders`) and `society` (trusted to the owning resource).

Current consumer: `bcc-society` (reference consumer). The development fixture
`feather-jobs-tests` is trusted only while DevMode is on.

## Database access

Only Jobs writes `feather_job_definitions`, `feather_job_grades`,
`feather_job_assignments`, `feather_job_character_state`, `feather_job_receipts`,
`feather_job_events`, `feather_job_outbox`, and `feather_job_schema_migrations`.
Consumers must use the exports above and never query these tables. Applied
migrations are checksummed and must not be modified; a mismatch fails readiness.

Mutations keep round trips low because every database statement costs about
one server tick. A freshly reserved receipt is not read back; event IDs (and the
assignment ID and timestamp) come from one query; events and outbox rows are
written with one multi-row insert each. `CreateAssignment` takes 9 queries (10
when it also selects the active assignment), down from 19.

## Remaining work

- **J5 consumer cutover:** bcc-doorlocks, bcc-craft and bcc-housing still read
  `character.job`/`jobGrade`. They are VORP and bcc-utils scripts that are not
  running on Feather yet, so each moves to `CheckAssignment` as part of its own
  Feather rewrite, as bcc-society did.
- **J6 legacy removal:** nothing remains on the Feather side. feather-character
  has no job fields or job mutators, and no Feather resource reads a character
  job. The legacy fields exist only in the VORP-era scripts above.
