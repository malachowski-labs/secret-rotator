# Glossary

Domain terms for secret-rotator. One name per concept; avoid the listed aliases.

## Rotator
A service that rotates one type of secret (for example Cloudflare tokens or database passwords). Each rotator has its own identity and its own Pub/Sub subscription, selected by a label on the secret.
_Avoid:_ worker, handler.

## Rotation
One end-to-end run for one secret: issue a new credential, write it as a new secret version, wait out the grace period, then revoke the previous credential.

## Target
The external system that issues and honors the credential (Cloudflare, a database, an SSH host).

## Grace period
How long the previous credential stays valid at the target after the new one is written. A value of 0 means the target invalidates the previous credential on issue, so there is nothing to revoke later.
_Avoid:_ cutover, TTL.

## Revocation probe
A safe, side-effect-free attempt to use the previous credential after it was revoked, to confirm it no longer works. Only run for targets where such a check cannot cause lockouts; otherwise the revoke result is trusted and logged as unverified.
_Avoid:_ revoke check, post-check.

## Sweeper
The single central job that finds rotations whose revoke is overdue and re-emits a revoke event to the matching rotator. It never touches target credentials.
