# Authorization Model

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/authorization-model.md`](https://github.com/iblai/ontology/blob/main/docs/authorization-model.md). This page is generated — edit it in the repository, not here.

> How iblai-ontology decides what a caller may access. See also the
> [identity deep-dive](/developer/guides/ontology-identity) and the [architecture overview](/developer/guides/ontology-architecture).

## Summary

Authorization is **role-based**, not per-subject-ownership based. Every request
carries a validated Entra ID JWT (authentication) and a platform-assigned role
(authorization). The gateway resolves *what that role can access* against
`roles.yaml` — toolsets, cache tables, and memory paths — and runs the tool if
the role permits it. The platform's service token and a person's own ibl.ai
credential authenticate the same way and end in the same place, a role; see
[identity.md](/developer/guides/ontology-identity#a-persons-own-iblai-credential).

Each inbound source (PeopleSoft, Canvas, Slate, Navigate, LDAP) is reached with
a **shared per-source service credential**. Credentials are isolated per source,
not per end user. This is intentional: the knowledge layer exists so that staff
agents (financial aid, advising, registrar) can query across many students'
records. "A counselor can look up any student by id" is the designed workflow,
not a flaw — see `identity.md` Step 3 for the canonical example.

## What is *not* enforced (by design)

- **Per-subject ownership on staff/analytics roles.** A role that is granted a
  student toolset may query any subject id within it. Cross-subject access is a
  property of the role grant, governed by which roles the ibl.ai platform
  assigns and by `roles.yaml`, not by comparing the caller to the subject.
- The real control against *unauthorized* cross-subject access is that a caller
  cannot choose their own role: the active role is derived from the validated
  token (see [#2138](https://github.com/iblai/ontology/issues/2138) and
  `roles.py::select_active_role`). Without that, no ownership policy could hold.

## What *is* self-scoped

The **self-service** role class (e.g. `Student` in
`roles.higher-ed.example.yaml`) is the one case with per-user scoping. It is
marked `self_service: true` and is restricted to the caller's own record on
both layers:

- **Memory** — paths are bound to the caller via `${USER_EMPLID}`, substituted
  with the caller's own id from the `IdentityMap` (`roles.py::resolve`). A
  student can only read `/ontology/students/by-id/<their-own-id>.md`.
- **Subject tools** — for a `self_service` role, any subject-identifier argument
  (`SUBJECT_ARG_KEYS` in `handlers.py`: `student_id`, `student`,
  `student_sis_id`, `emplid`) must equal the caller's own subject id, and the
  caller must have a known id (fail-closed). This keeps the tool layer
  consistent with the memory layer — a student cannot read another student's
  enrollment/aid by passing a different `student_id`.

Non-`self_service` roles are unrestricted by this check.

## Toolsets that change data

Out of the box every tool reads. An agent can change a record only through a tool
an administrator defined for that one change, and one rule carries it: **a toolset
that writes is granted by name, never by a wildcard.**

- A toolset is marked `write: true` in `tools.yaml` (the switch "These tools can
  change data" in the console). `mcp_toolsets: ["*"]` is every toolset *without*
  the mark. A role holds a marked one only when it names it, alone or beside the
  wildcard: `["*", "registrar-writes"]`. In the platform that is the resource
  `/custom/ontology/toolsets/registrar-writes/` beside `/custom/ontology/toolsets/`;
  a grant on the root alone never includes a toolset that writes.
- **What counts as a write** (`toolbox_compat.py::writing_tools`) fails closed. A
  tool reads only when the gateway can tell that it does: an `http` tool whose
  method is `GET` or `HEAD`, or a `*-sql` tool with one fixed statement that starts
  with `SELECT`, `WITH`, `SHOW`, `DESCRIBE`, `VALUES` or `TABLE`, holds no `;` and
  no write word (`INSERT`, `UPDATE`, `DELETE`, `MERGE`, `INTO`, `TRUNCATE`, `DROP`,
  `ALTER`, `CREATE`, `GRANT`, `CALL`, `EXEC`, `COPY`, `LOCK`, …). Everything else is
  a write: `*-execute-sql` tools, `templateParameters`, any other type. And **every
  tool on a source one tool writes through is a write**, so a read can never run
  on the write account. It is a text scan, not a parser: a write word inside a
  string makes a read count as a write (the refusal says which word), and a
  function that writes inside a `SELECT` is not seen, which is why a read source
  keeps its read-only account.
- **Three guards, each on its own.** The validator refuses to save a `tools.yaml`
  with a writing tool in an unmarked toolset. The gateway never lends a writing
  tool from an unmarked toolset, whatever the file says (a hand-edited or merged
  file skips the validator). And `tools/list` marks such a tool with MCP's
  `readOnlyHint: false` / `destructiveHint: true`, so a client may ask its user.
- **Nothing runs a write unattended or unrecorded.** A sync schedule cannot name a
  writing tool (refused when saved and when run); the console's Test button and
  `ontology mcp test` refuse one; and the gateway writes the audit row *before* the
  call. If that row cannot be written, the call is not made.

To add one: a second source with the write account's `${VAR}` references, a tool
whose statement is the `UPDATE` (or an `http` tool with `method: POST`), a toolset
with the mark, `ontology mcp build`, a Toolbox restart, then the role's switch on
the Access page. [components/01-mcp-inbound.md](https://github.com/iblai/ontology/blob/main/docs/components/01-mcp-inbound.md#tools-that-change-data)
has the YAML.

## Tools held by name

A role can also hold one tool by its own name (`mcp_tools`), beside its toolsets: this is
what a switch on the Access page writes when it stands for less than a whole toolset.

- The gateway lends a tool when one of the role's toolsets lends it, as above, or when the
  role names it. A tool held by name is lent as it is, whether it reads or changes data: the
  name is the grant, as a marked toolset's name is. `*` never names a tool.
- In the platform it is the resource `/custom/ontology/tools/<name>/` of the role's policy,
  counted when the role holds `Ontology/Tools/action`. A grant on the root, or on
  `/custom/ontology/tools/` alone, lends no tool.
- **A grant to read never becomes one to change.** When a save of `tools.yaml` removes a
  tool, or turns one that reads into one that changes data, the tool is taken from every role
  that holds it by name before the save stands; if the roles cannot be written, the file is
  put back.
- **What the switches do.** On a card of the Access page, Read is every tool under it that
  only reads and Write every tool under it that changes data: a tool is itself, a toolset its
  tools, a source the tools that use it, a system its own tools, and a system reached over a
  web API also has one switch per HTTP method. On adds the missing tools by name; on a
  toolset's own card a role that ends up with all it lends holds the toolset, so a tool added
  to it later comes along. Off takes the tools away however they were held: a toolset that
  lent one is replaced by its other tools, by name, and `*` by the read-only toolsets it stood
  for.
- **How far a role reaches** (`access` on `GET /roles`) is worked out from this: `read_write`
  when a tool that changes data is within its reach and something it can read is too, `write`
  or `read` when only one is, `none` when it reaches nothing. Reading is a tool that only
  reads, a memory path or a cache table. The console shows it after the role's name.

A connector can ship its operations instead. Connecting Twenty CRM adds the toolset
`<system>-writes`, already marked, with ten tools of type `twenty` that the gateway
runs itself with the system's own key: create and update, no delete. The same
guards apply, since any type the gateway cannot tell to be a read is a write. The
key's own permissions in Twenty are the outer limit
([components/02-knowledge-materialization.md](https://github.com/iblai/ontology/blob/main/docs/components/02-knowledge-materialization.md#an-application-read-over-its-api)).

## Querying the cache

`describe-ontology-cache` and `query-ontology-cache` are the gateway's own tools.
They are offered to a role that may query at least one table (`cache_tables`), not
through a toolset: a role given a CRM's tables is not handed every other tool of a
shared toolset with them. The first lists the tables the role may query, with their
columns; the second runs one `SELECT` on them in a read-only transaction.

## Row limit

`concurrency_limits.max_records_per_query` caps what `query-ontology-cache`
returns to the role, whatever `limit` the agent passes. It must be a whole number
above zero; absent is no limit. The console offers 100, 1,000, 10,000 and "No
limit", and a new role starts at 1,000. In the platform store it is also a resource
of the role's policy (`/custom/ontology/limits/rows/<n>/`), because a caller with an
ibl.ai credential is judged by its grants, which carry no role name: of several
roles' limits the largest applies, and an administrator of the platform has none.

## Adding a role

- Staff / analytics / admin role → omit `self_service` (defaults to `false`);
  the role's toolsets and cache tables define its reach.
- A role that must only ever see its own holder's record → set
  `self_service: true` and template memory paths with `${USER_EMPLID}`.

## Audit

Every access, allowed or denied, is written to `audit_log` keyed by the token
`jti` (`middleware.py::write_audit`), so any data access is traceable from login
to the specific tool call. A row's `details` say whether the tool itself failed
(`isError`). A call to a tool that changes data has a second row, written first:
`mcp.tools/call.write`, with the arguments it was given (the first 4,000
characters).
