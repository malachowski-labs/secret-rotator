# Repository guardrails

Settings are applied by hand in the GitHub repository; they are not managed as code yet.

## Default branch

A repository ruleset on the default branch (Settings → Rules → Rulesets) enforces:

- Changes arrive only through pull requests. No approving review is required (solo-maintained).
- Force pushes and branch deletion are blocked.
- No bypass actors, so the rules also apply to administrators.

**Required status checks: pending.** They are added to the ruleset when the first CI workflow exists.

## Other protections

- Secret scanning and push protection are enabled (Settings → Code security).
- Dependabot alerts are enabled (Settings → Code security).
- `.github/CODEOWNERS` assigns the maintainer as default owner. Code-owner review is not required by the ruleset.

## Changing them

Edit the ruleset or the Code security toggles in the repository settings, and update this page in the same pull request. Revisit managing them as code when the OpenTofu module work lands.
