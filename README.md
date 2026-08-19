# platform-workflows

Versioned, platform-owned CI logic for services on the
[`terasky-platform-ha`](https://github.com/itayna/terasky-platform-ha) paved road.
Services call these workflows; they do not copy them.

## Using it

A service repository's entire `.github/workflows/delivery.yml`:

```yaml
name: delivery
on:
  push:
    branches: [main]
jobs:
  delivery:
    uses: itayna/platform-workflows/.github/workflows/java-service-delivery.yml@v1
    with:
      service-name: my-service
    secrets:
      PLATFORM_REPO_DEPLOY_KEY: ${{ secrets.PLATFORM_REPO_DEPLOY_KEY }}
```

`make onboard` in the platform repository writes this file for you.

## `java-service-delivery.yml`

Maven build and test → Trivy filesystem scan (fails on CRITICAL) → multi-arch
build and push to GHCR → Trivy image scan (fails on CRITICAL) → SPDX SBOM
attached to the image → keyless cosign signature → commit the new tag into
`environments/dev/<service>/kustomization.yaml` in the platform repository, which
Argo CD then syncs.

| Input | Default | |
|---|---|---|
| `service-name` | — | required; image name and manifest directory |
| `java-version` | `25` | Temurin JDK |
| `platform-repo` | `itayna/terasky-platform-ha` | GitOps repo the dev tag is written to |
| `registry` | `ghcr.io` | |

| Secret | |
|---|---|
| `PLATFORM_REPO_DEPLOY_KEY` | SSH deploy key with write access to the platform repo |

Prod is never touched from here — promotion is a separate, manually dispatched
workflow in the platform repository that re-verifies the signature and opens a PR.

## Versioning

| Ref | Meaning |
|---|---|
| `@v1` | floating major. Moves as `v1.x` releases land. What services should use. |
| `@v1.0` | immutable. For a service that must opt out of a change temporarily. |
| `@main` | unversioned. Do not use; it is the platform's own test surface. |

**Adding a mandatory step to every service** (the case this repository exists for):

1. Commit the step here.
2. Tag `v1.1`, then move `v1` to it:
   `git tag -f v1 v1.1 && git push -f origin v1`
3. Every service on `@v1` picks it up on its next push. Nothing changes in any
   service repository, and the platform team edits no service's CI.

Services pinned to `@v1.0` do not pick it up. The platform repository's
compliance scorecard reports each service's pinned version, so "who has not
upgraded" is a table, not an investigation.

**Breaking changes** get `v2` and a new floating tag; `v1` keeps working. A
service moves by editing the one `uses:` line, which is a reviewable diff in the
service's own history.

## Signing identity

With a reusable workflow the Fulcio certificate identity is *this* workflow's
ref, not the caller's. The cluster admission policy therefore trusts one identity
for all services rather than one rule per service — which is what makes
onboarding a service need zero platform-team action. What the signature proves is
"built by version v1.x of the platform pipeline", with the service repository
recorded in the certificate extensions and the SBOM. Trade-off written up in
ADR-0008 in the platform repository.
