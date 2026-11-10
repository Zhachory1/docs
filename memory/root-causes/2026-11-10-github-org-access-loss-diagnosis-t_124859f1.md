---
kind: root-cause
source: t_124859f1
profile: ralph-author
date: 2026-11-10
---

# A sandboxed agent that suddenly 404s on every organization repository has lost OAuth-app org approval, not SAML SSO authorization

## Symptom

A long-running agent sandbox that had been reading and writing organization repositories
all day began returning `HTTP 404 Not Found` for every organization-owned repository,
while the same token kept working normally for personal repositories.

## How to tell the two causes apart

Revoked SSO authorization and revoked OAuth-app approval both look like "my token stopped
working". One request separates them:

| Probe | SSO authorization revoked | OAuth app approval revoked |
|---|---|---|
| Repository request | `403` with an `X-GitHub-SSO: required` header | bare `404`, no SSO header |
| `GET /user/orgs` | lists the organizations | `[]`, even with `read:org` granted |
| `GET /user/repos?affiliation=organization_member` | non-empty | `0` |

The decisive signal is the pair: an empty `/user/orgs` while `read:org` is still present in
`X-OAuth-Scopes`, together with a 404 that carries no SSO header. GitHub returns 404 rather
than 403 for repositories a token may not know exist, so the status code alone is not enough.

## Things that look like the cause and are not

- Scopes: unchanged and sufficient (`repo` present in `X-OAuth-Scopes`).
- Rate limiting: check `GET /rate_limit`. A throttled token returns 403, not 404.
- Network, or a deleted organization: `GET /orgs/<name>` still returns `200`, so name
  resolution and connectivity are fine while that organization's repositories 404.
- The one repository you care about: check
  `GET /user/repos?affiliation=organization_member`. If that returns zero repositories
  anywhere, the problem is the token's organization reach, not a single repository.
- A missing git credential helper: test the same repository over HTTPS with the token
  inlined. `Repository not found` on an organization repository, while a personal
  repository returns refs, confirms authorization rather than transport.

## Why it matters for agents

An agent cannot repair this. It needs an organization owner to re-approve the OAuth
application (Organization settings, Third-party access, OAuth app access policy) or to
restore the member's organization membership. An agent that hits this should run the
probes above, state which of the two fixes is needed, and stop rather than retry.

## Protect the deliverable before the sandbox is recycled

Finished work can be stranded twice over when access is lost. A pushed branch becomes
unreachable along with everything else, and a scratch workspace does not survive the run:
a later attempt can land in a fresh container where the clone, the patch file and the
working tree are all gone.

Attach the finished artifact to its tracking card as soon as it exists. A host-side
attachment store is not necessarily readable from inside the agent's sandbox, so an
attachment preserves the artifact for a human but does not guarantee the next sandboxed
worker can read it. Put content a downstream worker must act on in a card comment, which
every worker can read, rather than only in a file.

## The cheap unblock people miss

When the branch is already pushed and only the pull request is missing, a human with
access can open it from the repository's branch-compare page in the web interface. That
needs no change to any access policy, and it is far less friction than re-granting
organization access just to let the agent run one command.
