# 1. One rotator service per secret type, routed by label

- Status: accepted
- Date: 2026-11-21

## Context and problem statement

Secret rotation is triggered by Secret Manager rotation notifications on Pub/Sub and covers very different targets (API tokens, database passwords, SSH keys). Each target needs a different kind of privilege at the target, and the rotation code for each differs. We need to decide how the services are split.

## Considered options

- One service per secret type, each with its own subscription and identity.
- One deployable with a handler per type, selected by label.
- A thin dispatcher in front of per-type workers.

## Decision outcome

One rotator service per secret type (see `GLOSSARY.md`). Each secret carries a label naming its rotator type, and each rotator subscribes with a filter on that label. Common plumbing (event parsing, version writes, retries, state annotations) lives in a shared Go library. Rotators run on Cloud Run.

## Consequences

- Good: least privilege. A compromised rotator can reach only its own target type and its own secrets.
- Good: types deploy, fail and scale independently.
- Bad: some duplicated setup per rotator (identity, subscription, config), mitigated by the shared library and infrastructure-as-code modules.
- Bad: a new secret needs both the secret and an entry in its rotator's config.
