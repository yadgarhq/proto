# Errata

Releases that changed documented behaviour of an existing RPC without a `!`
(breaking) Changelog bullet, so they shipped as a minor or patch release. Tags
are never rewritten (ADR-1167). Each entry says what changed; the version it
shipped as stays. Rule: ADR-1258.

## v1.24.0 (2026-10-24)

Should have been a major release. AddTeamMember and RemoveTeamMember always answer FAILED_PRECONDITION, because teams are declared in GitOps.

## v1.22.0 (2026-10-23)

Should have been a major release. Task titles are limited to 128 Unicode code points. CreateTask and EditTask refuse a longer title with INVALID_ARGUMENT.

## v1.21.1 (2026-10-21)

Should have been a major release. A replayed RedeemEnrolment returns the credential epoch recorded at first delivery, not the current epoch. A replay after a later bump is refused with FAILED_PRECONDITION.

## v1.21.0 (2026-10-21)

Should have been a major release. RevokeCredential requires `user_id`. iam and iam-db each accept one caller identity only and answer PERMISSION_DENIED to any other. Login and redemption credentials now expire.

## v1.20.1 (2026-10-19)

Should have been a major release. RegisterProject refuses, with PERMISSION_DENIED, a `local/<seg>/...` path whose segment is not the caller's user id. ListProjects is answered only for the gateway.

## v1.17.0 (2026-10-17)

Should have been a major release. RegisterProject accepts only personal-class (`local/...`) paths and refuses an organisation-class path with FAILED_PRECONDITION.

## v1.13.0 (2026-09-12)

Should have been a major release. RegisterProject requires `source_repo` for an organisation path, and the store refuses an organisation row without it.

## v1.7.1 (2026-09-02)

Should have been a major release. IssueEnrolment refuses a replayed idempotency key with INVALID_ARGUMENT. Before, a replay returned the original token.

## v1.6.0 (2026-09-01)

Should have been a major release. A reused idempotency key that carries a different request is refused with INVALID_ARGUMENT. Before, it was replayed with the original outcome.
