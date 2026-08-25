# Authorization Model

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/authorization-model.md`](https://github.com/iblai/ontology/blob/main/docs/authorization-model.md). This page is generated — edit it in the repository, not here.

> How iblai-ontology decides what a caller may access. See also the
> [identity deep-dive](/developer/guides/ontology-identity) and the [architecture overview](/developer/guides/ontology-architecture).

## Summary

Authorization is **role-based**, not per-subject-ownership based. Every request
carries a validated Entra ID JWT (authentication) and a platform-assigned role
(authorization). The gateway resolves *what that role can access* against
`roles.yaml` — toolsets, cache tables, and memory paths — and runs the tool if
the role permits it.

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

## Adding a role

- Staff / analytics / admin role → omit `self_service` (defaults to `false`);
  the role's toolsets and cache tables define its reach.
- A role that must only ever see its own holder's record → set
  `self_service: true` and template memory paths with `${USER_EMPLID}`.

## Audit

Every access, allowed or denied, is written to `audit_log` keyed by the token
`jti` (`middleware.py::write_audit`), so any data access is traceable from login
to the specific tool call.
