---
title: "Automated Dependency Updates on Self-Hosted GitLab with Renovate"
date: 2026-04-12T09:32:39
lastmod: 2026-04-12T09:32:39
aliases:
  - /2026/04/12/renovate-self-hosted-gitlab/
description: "How to set up Renovate on a self-hosted GitLab CE instance to automatically discover all repos in a group, raise dependency update MRs, and hold them for 7 days before opening."
tags: ["gitlab", "ci-cd", "renovate"]
---

If you’re running a self-hosted GitLab CE instance with a bunch of repos, keeping dependencies and Docker image tags up to date across all of them is a pain. Renovate solves this — it’s an open-source tool that opens MRs automatically when new versions are available. This post walks through setting it up from scratch on GitLab CE, including autodiscovery across a whole group and a 7-day hold before any MR gets raised.

## How Renovate Works on GitLab

Unlike GitHub Actions where Renovate can run natively, on GitLab you run Renovate yourself as a scheduled CI pipeline. A dedicated bot user authenticates with your GitLab API, discovers repos it has access to, and opens MRs for any outdated dependencies it finds. It’s stateless — each run scans everything and exits. State is tracked via open MRs and branch names.

## 1. Create a Bot User and Personal Access Token

Renovate needs a dedicated GitLab user to open MRs as. Create a new user (e.g. `renovate-bot`) and generate a Personal Access Token with the **api** scope. This is the only scope Renovate needs — and it must be `api`, not `read_api`. Renovate needs to create branches, open MRs, and post comments.

A common mistake here is using a Project Access Token or a Deploy Token. Neither will work for instance-wide autodiscovery. It must be a Personal Access Token belonging to the bot user.

Once the user is created, add it as a **Developer** member of the GitLab group containing your repos. Renovate’s autodiscovery only finds repos the bot user has access to — if it can’t see them, it won’t touch them.

## 2. Create a renovate-runner Repo

Create a new repo (e.g. `lab/renovate-runner`) that will house the Renovate config and scheduled pipeline. This repo is the controller — it’s not a repo Renovate manages, it’s the repo Renovate runs *from*.

You need two files: a `config.js` for Renovate’s global settings, and a `.gitlab-ci.yml` to run it on a schedule.

### config.js

```
module.exports = {
  platform: 'gitlab',
  endpoint: 'https://your-gitlab.example.com/api/v4',

  autodiscover: true,
  autodiscoverFilter: ['lab/**'],

  onboarding: true,
  requireConfig: 'optional',

  gitAuthor: 'Renovate Bot <renovate-bot@example.com>',

  // Hold all updates for 7 days after release
  minimumReleaseAge: '7 days',

  packageRules: [
    {
      matchManagers: ['dockerfile', 'docker-compose'],
      minimumReleaseAge: '7 days',
    },
  ],
};
```

A few things worth calling out here. The `autodiscoverFilter` uses GitLab’s `namespace/project` path format. **This must be a group path, not a personal namespace.** If your repos live in a personal namespace (your own username), autodiscovery will return zero results — GitLab’s API treats personal namespaces differently from groups for membership queries. Create a group and work from there.

The `minimumReleaseAge: '7 days'` setting holds Renovate back from raising an MR until a release is at least 7 days old. This gives time for a bad release to be caught and yanked before you’re automatically updating to it. Renovate detects the new version immediately — it just waits before acting on it.

### .gitlab-ci.yml

```
renovate:
  image: renovate/renovate:latest
  variables:
    RENOVATE_PLATFORM: gitlab
    RENOVATE_ENDPOINT: https://your-gitlab.example.com/api/v4
    LOG_LEVEL: debug
  script:
    - renovate
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: '$CI_PIPELINE_SOURCE == "web"'
```

Notice there’s no `RENOVATE_TOKEN: $RENOVATE_TOKEN` mapping in the variables block. This is intentional. GitLab CI/CD variables are already in the environment — if you explicitly map a masked variable to itself, it can expand to an empty string before the container sees it, causing a 401 from the GitLab API. Set `RENOVATE_TOKEN` as a masked CI/CD variable in the runner repo settings and let it flow through naturally.

## 3. Set the CI/CD Variable

In the `renovate-runner` repo, go to **Settings → CI/CD → Variables** and add:

- **Key:** `RENOVATE_TOKEN`
- **Value:** the PAT you created for the bot user
- **Masked:** yes
- **Protected:** only if your main branch is protected (otherwise the variable will be silently empty on unprotected pipelines)

## 4. Schedule It

Go to **CI/CD → Schedules** in the runner repo and create a schedule. Every 1–4 hours is reasonable depending on how many repos you have. Renovate is fast on small groups and progressively slower as the repo count grows — start with every 2 hours and adjust.

The `web` rule in the CI config also lets you trigger a run manually, which is useful when you first get things working and want to verify without waiting for the schedule.

## 5. Onboarding

With `onboarding: true`, the first time Renovate visits a repo it opens a **“Configure Renovate”** MR containing a default `renovate.json`. Merging it signals to Renovate that the repo is opted in. With `requireConfig: 'optional'`, Renovate will still process repos that don’t have a config file — useful while you’re rolling out across a group.

## Troubleshooting

A few issues that come up reliably:

**401 Unauthorized.** Check the PAT has the `api` scope (not just `read_api`), and that you’re not self-referencing the token variable in the CI config as described above.

**Discovered 0 projects.** The bot user has no access to anything, or you’re using a personal namespace in `autodiscoverFilter` instead of a group path. Add the bot user as Developer on the group and switch to a group path.

**MRs not appearing after 7 days.** Renovate needs to be able to determine the release timestamp from the registry. Public registries work fine. For private registries you’ll need to configure `hostRules` with credentials so Renovate can query the registry API to check release dates.

## What’s Next

With Renovate running, the natural next step is adding container scanning to the MRs it raises — so you’re not just updating to the latest image, you’re updating to a latest image that’s been scanned for CVEs and supply chain issues before it merges. A follow-up post will cover adding Trivy and Grype as parallel CI jobs using a shared template, so you only have to maintain the scanning logic in one place across all your repos.
