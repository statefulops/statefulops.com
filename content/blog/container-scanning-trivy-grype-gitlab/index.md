---
title: "Container Scanning on GitLab MRs with Trivy and Grype"
date: 2026-04-12T17:53:26
lastmod: 2026-04-12T17:53:26
aliases:
  - /2026/04/12/container-scanning-trivy-grype-gitlab/
description: "How to add parallel Trivy and Grype container scanning to GitLab merge requests using a shared CI template, so every Renovate-raised image update gets scanned for CVEs and supply chain issues before it merges."
tags: ["gitlab", "ci-cd", "security"]
---

In the [previous post](/blog/renovate-self-hosted-gitlab/) we got Renovate running on self-hosted GitLab CE, automatically discovering repos and raising MRs when new image versions are available. That handles the update workflow — but it doesn’t tell you whether the new image is actually safe to run. This post adds container scanning to the picture: every MR that touches a Docker Compose file gets scanned by both Trivy and Grype before it can merge.

## Why Two Scanners?

Trivy and Grype cover overlapping but not identical ground. Trivy is strong on OS-level CVEs and is fast to run. Grype has good coverage of supply chain compromises and malicious packages — things that might not show up as a CVE but indicate something wrong with the image’s provenance. Running both in parallel costs almost nothing extra and gives you better signal than either alone.

Neither of these is a malware scanner in the antivirus sense. If you need filesystem-level malware detection, ClamAV or YARA rules are worth adding on top. But for a GitOps workflow where Renovate is proposing image updates, CVE and supply chain coverage is where most of the risk sits.

## The Approach

Rather than adding scanning logic to every repo individually, we use a shared CI template in a `ci-templates` repo. Any compose repo can adopt it with two lines. The template runs three jobs on every MR:

- **resolve-images** — diffs the MR against the target branch, finds changed compose files, extracts unique image references, and writes them to `images.txt`
- **trivy-scan** — reads `images.txt` and scans each image with Trivy
- **grype-scan** — reads the same `images.txt` and scans each image with Grype

Trivy and Grype run in parallel via GitLab’s `needs:` keyword, both depending on `resolve-images`. The image list is resolved once and shared as a job artifact — there’s no duplicated compose-parsing logic.

Scoping to changed files only matters here. A repo with 20 compose files across different directories would be slow and noisy if every MR scanned everything. Because Renovate only touches the specific compose file containing the updated image, the diff approach means you’re scanning exactly what changed — usually one or two images per MR.

## The Shared Template

Create a `ci-templates` repo in your GitLab group and add a `container-scan.yml`. Here’s the full template:

```
stages:
  - container-scan

.resolve-images:
  stage: container-scan
  image: alpine:latest
  variables:
    SCAN_SEVERITY: "HIGH,CRITICAL"
    SCAN_IGNORE_UNFIXED: "true"
    SCAN_FAIL_ON_VULN: "true"
  before_script:
    - apk add --no-cache yq git
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  script:
    - |
      git fetch origin "$CI_MERGE_REQUEST_TARGET_BRANCH_NAME" --depth=20

      CHANGED_COMPOSE=$(git diff --name-only \
        "origin/${CI_MERGE_REQUEST_TARGET_BRANCH_NAME}...HEAD" \
        | grep -E '(docker-compose|compose)\.(yml|yaml)$' || true)

      if [ -z "$CHANGED_COMPOSE" ]; then
        echo "No compose files changed — skipping scan."
        touch images.txt
        exit 0
      fi

      echo "$CHANGED_COMPOSE" \
        | xargs -I{} yq e '.services[].image // ""' {} \
        | grep -v '^$' \
        | grep -v '^null$' \
        | sort -u > images.txt

      cat images.txt
  artifacts:
    paths:
      - images.txt
    expire_in: 1 hour

.trivy-scan:
  stage: container-scan
  image:
    name: ghcr.io/aquasecurity/trivy:latest
    entrypoint: [""]
  variables:
    SCAN_SEVERITY: "HIGH,CRITICAL"
    SCAN_IGNORE_UNFIXED: "true"
    SCAN_FAIL_ON_VULN: "true"
    TRIVY_INSECURE: "false"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  script:
    - |
      if [ ! -s images.txt ]; then
        echo "No images to scan — skipping Trivy."
        exit 0
      fi

      TRIVY_FLAGS="--no-progress --format template --template @/contrib/junit.tpl"
      TRIVY_FLAGS="$TRIVY_FLAGS --severity $SCAN_SEVERITY"
      [ "$SCAN_IGNORE_UNFIXED" = "true" ] && TRIVY_FLAGS="$TRIVY_FLAGS --ignore-unfixed"
      [ "$SCAN_FAIL_ON_VULN"   = "true" ] && TRIVY_FLAGS="$TRIVY_FLAGS --exit-code 1"

      mkdir -p reports/trivy
      OVERALL_EXIT=0

      while IFS= read -r IMAGE; do
        [ -z "$IMAGE" ] && continue
        SAFE_NAME=$(echo "$IMAGE" | tr '/: ' '---')
        # shellcheck disable=SC2086
        trivy image $TRIVY_FLAGS \
          --output "reports/trivy/${SAFE_NAME}.xml" \
          "$IMAGE" || OVERALL_EXIT=1
      done < images.txt

      exit $OVERALL_EXIT
  artifacts:
    when: always
    reports:
      junit: reports/trivy/*.xml
    paths:
      - reports/trivy/
    expire_in: 7 days

.grype-scan:
  stage: container-scan
  image: alpine:latest
  variables:
    SCAN_SEVERITY: "HIGH,CRITICAL"
    SCAN_IGNORE_UNFIXED: "true"
    SCAN_FAIL_ON_VULN: "true"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  before_script:
    - apk add --no-cache curl
    - curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
    - curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/templates/junit.tmpl -o /usr/local/share/grype-junit.tmpl
  script:
    - |
      if [ ! -s images.txt ]; then
        echo "No images to scan — skipping Grype."
        exit 0
      fi

      GRYPE_FAIL_ON="high"
      echo "$SCAN_SEVERITY" | grep -qi "MEDIUM" && GRYPE_FAIL_ON="medium"
      echo "$SCAN_SEVERITY" | grep -qi "LOW"    && GRYPE_FAIL_ON="low"

      GRYPE_FLAGS="--output template -t /usr/local/share/grype-junit.tmpl"
      [ "$SCAN_IGNORE_UNFIXED" = "true" ] && GRYPE_FLAGS="$GRYPE_FLAGS --only-fixed"
      [ "$SCAN_FAIL_ON_VULN"   = "true" ] && GRYPE_FLAGS="$GRYPE_FLAGS --fail-on $GRYPE_FAIL_ON"

      mkdir -p reports/grype
      OVERALL_EXIT=0

      while IFS= read -r IMAGE; do
        [ -z "$IMAGE" ] && continue
        SAFE_NAME=$(echo "$IMAGE" | tr '/: ' '---')
        # shellcheck disable=SC2086
        grype "$IMAGE" $GRYPE_FLAGS \
          --file "reports/grype/${SAFE_NAME}.xml" || OVERALL_EXIT=1
      done < images.txt

      exit $OVERALL_EXIT
  artifacts:
    when: always
    reports:
      junit: reports/grype/*.xml
    paths:
      - reports/grype/
    expire_in: 7 days

resolve-images:
  extends: .resolve-images

trivy-scan:
  extends: .trivy-scan
  needs:
    - job: resolve-images
      artifacts: true

grype-scan:
  extends: .grype-scan
  needs:
    - job: resolve-images
      artifacts: true
```

## A Few Things Worth Calling Out

**Trivy comes from GHCR, not Docker Hub.** The `aquasec/trivy:latest` tag on Docker Hub doesn’t exist — Trivy’s canonical public image is `ghcr.io/aquasecurity/trivy:latest`. This will cause a manifest not found error if you use the Docker Hub reference.

**Grype’s official image is distroless.** GitLab’s runner needs a shell to exec into the container to run the job script. The Grype image has no shell at all, so the job fails immediately with `exec: "sh": executable file not found in $PATH`. The fix is to use Alpine and install Grype via the official install script at runtime. It adds a few seconds per job but is straightforward and reliable.

**Grype dropped native JUnit output.** The `--output junit` flag no longer exists in recent Grype versions. Instead, Grype supports a `template` output format where you supply a Go template file. The Anchore team maintain an official JUnit template in the Grype repo, so the `before_script` downloads it alongside the Grype binary. The template approach is actually more flexible — you could swap in a custom template if you needed different output.

**Artifacts must be in the project directory.** GitLab artifacts have to be within `$CI_PROJECT_DIR`. Writing `images.txt` to `/tmp` seems natural but the artifact upload silently fails with a warning and the downstream jobs get an empty file. Write it to the working directory and reference it as a relative path in the `artifacts:` block.

## Adopting the Template

Any repo with compose files can adopt the template by adding a `.gitlab-ci.yml`:

```
include:
  - project: 'lab/ci-templates'
    ref: main
    file: '/container-scan.yml'
```

That’s it for the default configuration. The jobs run automatically on every MR that touches a compose file. To override behaviour for a specific repo, extend the concrete jobs:

```
include:
  - project: 'lab/ci-templates'
    ref: main
    file: '/container-scan.yml'

trivy-scan:
  variables:
    SCAN_SEVERITY: "MEDIUM,HIGH,CRITICAL"
    SCAN_FAIL_ON_VULN: "true"
    TRIVY_USERNAME: $MY_REGISTRY_USER
    TRIVY_PASSWORD: $MY_REGISTRY_PASSWORD

grype-scan:
  variables:
    SCAN_FAIL_ON_VULN: "false"   # advisory only while baselining
```

A useful pattern when rolling out across a group for the first time is to set `SCAN_FAIL_ON_VULN: "false"` on the Grype job initially. Grype’s supply chain checks can be noisy on older images and it’s worth baselining what it finds before making it a hard gate. Trivy can be blocking from day one — its CVE output is more predictable.

## Blocking Merges on Findings

For the scan to actually block a merge, two things need to be true. First, `SCAN_FAIL_ON_VULN` must be `true` so the job exits non-zero on findings. Second, the repo needs **Settings → Merge requests → Pipelines must succeed** enabled. Without that setting, a failing pipeline is visible in the MR but doesn’t prevent merging.

This also interacts with Renovate’s `automerge` setting. If you’ve enabled automerge for patch updates, Renovate will only automerge if all required status checks pass — so a failing scan will hold it back correctly without any extra config.

## Ignoring Known CVEs

Both scanners will find CVEs in base images you don’t control and can’t fix. To keep results actionable, `SCAN_IGNORE_UNFIXED: "true"` is on by default — this filters out CVEs where no fix is available upstream, which cuts the noise significantly.

For CVEs you’ve assessed and accepted, add a `.trivyignore` file at the repo root:

```
# .trivyignore
# CVE-ID        expiry        comment
CVE-2023-1234   2024-12-01    # upstream fix not yet released
CVE-2023-5678                 # not exploitable in our context
```

Trivy picks this up automatically. Grype has an equivalent via a `.grype.yaml` ignore list if you need per-CVE suppression there too.

## The Full Picture with Renovate and Portainer

With both posts in place, the end-to-end flow for a GitOps Portainer setup looks like this:

- Renovate detects a new image tag and waits 7 days
- Renovate opens an MR updating the tag in the relevant compose file
- The CI pipeline triggers: `resolve-images` diffs the MR and extracts the changed image, then `trivy-scan` and `grype-scan` run in parallel
- Results appear as a JUnit test report in the MR UI — pass or fail per image, per CVE
- If both scanners pass, the MR can be merged (or automerged if configured)
- Portainer detects the change on the main branch and redeploys the stack with the new image

The 7-day hold and the scanning gate together mean you’re not deploying a brand new image immediately and you’re not deploying one with known exploitable CVEs. It’s not a complete supply chain story — digest pinning and SBOM generation are worth adding if you want to go further — but it’s a solid baseline for a self-hosted GitOps setup without any paid tooling.
