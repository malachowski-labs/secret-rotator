# 2. Rotation lifecycle, state and failure recovery

- Status: accepted
- Date: 2026-11-21

## Context and problem statement

A rotation spans the target, Secret Manager and a delayed revoke, with no shared transaction. Pub/Sub delivers at least once, and most targets need an overlap between the old and new credential. We need a lifecycle that tolerates retries and partial failure without adding heavy infrastructure.

## Considered options

- State as annotations on the secret, a delayed-revoke task, and a central sweeper.
- A separate state store with a sweeper over it.
- A workflow engine owning the whole lifecycle.

## Decision outcome

Each rotation runs: issue a new credential, write it as a new secret version, verify it, schedule the revoke after the grace period, revoke, confirm the revoke where a safe probe exists, then disable and destroy the old version with the secret's version destroy TTL (7 days, global default).

- Retiring the old version is a single destroy call: on a secret with a version destroy TTL, Secret Manager disables the version immediately and schedules its permanent destruction after the TTL. During the delay an administrator can restore it by enabling it.
- Secret Manager advances the secret's next rotation time when it sends the rotation notification, whether or not the rotation succeeds. A failed rotation therefore waits a full rotation period unless a human re-drives it.
- Revocation probe: where the target offers a safe, side-effect-free check, the rotator tries the old credential after the revoke, allowing for propagation delay. If it still works, the rotator retries the revoke with backoff and alerts. The old version stays enabled (a disabled version cannot be read) and its destroy TTL does not start until the probe confirms the credential is dead. Revoked-but-unconfirmed is an overdue phase for the sweeper and alerts like a lost revoke.
- Where no safe probe exists (for example lockout risk), the revoke result is trusted, the outcome is logged as unverified, and the old version is destroyed as usual. Destroying the version removes only our copy; it does not revoke anything at the target.

- Every step is idempotent and safe to replay. Writing the secret version is the pivot point: before it a failure compensates, after it recovery only goes forward.
- A grace period of 0 means the target invalidates on issue and there is no revoke step. In that case recovery is to re-issue.
- Progress is kept as annotations on the secret (rotation ID, phase, revoke-after). Routing uses labels. Target-specific detail stays in the rotator's infrastructure-managed config, referenced by an opaque target name.
- Verification is mandatory and uses the new credential. A failed verify blocks the revoke. This protects admin credentials that rotate themselves.
- Where a target supports keyless authentication, rotators use it. Otherwise the admin credential is itself a rotated secret; only its first value is placed by hand.
- A single central sweeper (Cloud Run on a schedule) finds revokes past their revoke-after time and re-emits a revoke event to the matching rotator by label. It is read-only plus publish-only and never touches target credentials. A lost revoke also raises an alert.
- Permanent rotation failures go to a dead-letter topic with a log-based alert and are re-driven manually. The sweeper does not re-drive rotations.
- The audit trail is structured logs; alerts are log-based.

## Consequences

- Good: no extra datastore; state sits next to the secret it describes.
- Good: the sweeper keeps least privilege and closes the lost-revoke gap.
- Bad: annotations are mutable and size-limited, and give no queryable history. Revisit with a state store if history becomes a requirement.
- Bad: a rotation that fails permanently stays unrotated until a human acts, up to the next scheduled rotation.
- Open: GitHub tokens may not be creatable through an API; scope is decided separately. The documentation describes generating a GitHub App private key only through the app's settings page.
- Open: delayed destruction of secret versions was announced as Preview in April 2024; confirm its current availability before the rotators depend on it.
