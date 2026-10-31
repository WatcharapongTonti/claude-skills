# claude-skills

Custom [Claude](https://claude.ai) skills for backend / platform engineering work.

A skill is a folder with a `SKILL.md` file: YAML frontmatter (`name`, `description`) plus instructions Claude follows when the skill is triggered.

## Skills

| Skill | What it does |
|---|---|
| [`trivy-cve-remediation`](skills/trivy-cve-remediation/SKILL.md) | Fixes CVEs reported by a Trivy scan (container image, filesystem, repo), CRITICAL first, then HIGH, MEDIUM, LOW. Triages findings, upgrades with the smallest safe change, checks for impact, rescans, and documents accepted risk. |

### trivy-cve-remediation

- **Severity tiers:** CRITICAL -> HIGH -> MEDIUM -> LOW/UNKNOWN. Each tier is fixed and verified by a rescan before the next starts; CRITICAL fixes can merge on their own.
- **Always Trivy `latest`:** pulls `aquasec/trivy:latest` (or the latest binary) before every scan. If a new Trivy version is released mid-work, it switches right away and re-baselines so before/after results come from the same version.
- **Fresh vulnerability DB** for every scan. `--skip-db-update` is used only on the scan step right after a successful `--download-db-only` in the same job.
- **CI scan stage:** makes sure the GitLab CI pipeline has an image-scan job using Trivy `latest` (`pull_policy: always`, `--scanners vuln`), and adds one if missing. The GitLab CI template rules (dev-only scan, `needs`, Harbor image path, Node builder image, approved Harbor base images) are built into the skill, so `gitlabci` is not needed.
- **CI Trivy DB download fix (`WARNING: Cache file does not exist` / `connection reset by peer`):** the cache warning is only a symptom. Runner job containers cannot reach public registries, GitLab saves the cache only after a successful job, and Trivy does not fall back to the next `--db-repository`. The skill downloads the DB through the Harbor Docker Hub proxy first (`$HARBOR_URL/docker_hub_proxy_cache/aquasec/trivy-db:2`), tries public sources in a shell loop, caches `.trivycache/`, then scans with `--skip-db-update`. It verifies the fix in CI (not locally), never treats `allow_failure: true` as the fix, and reports it as blocked for the network team if every source fails.
- **No-impact checks:** baseline tests, dependency-tree diff, image config/package diff, smoke test.
- **Report:** before/after counts per severity, the Trivy version used, fixes, blocked items, follow-ups.

## Install

**Claude Code**

```bash
git clone https://github.com/WatcharapongTonti/claude-skills.git
# personal (all projects)
cp -r claude-skills/skills/trivy-cve-remediation ~/.claude/skills/
# or per project
cp -r claude-skills/skills/trivy-cve-remediation <project>/.claude/skills/
```

**Claude app (claude.ai / desktop)**

Zip the skill folder and upload it as a custom skill:

```bash
cd claude-skills/skills && zip -r trivy-cve-remediation.zip trivy-cve-remediation
```

## Usage

Ask Claude in plain language, for example:

- "Fix the CVEs from this Trivy report"
- "Scan `harbor.example.com/team/app:1.2.3` with Trivy and fix the CRITICAL and HIGH findings"
- "Our image has vulnerabilities, clean them up"
- "The image-scan job fails with `Cache file does not exist` and `failed to download vulnerability DB`"

## Repo layout

```
skills/
  <skill-name>/
    SKILL.md        # required: frontmatter + instructions
    scripts/        # optional helper scripts
    references/     # optional docs loaded on demand
```

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md` with `name` and `description` frontmatter.
2. Add a row to the Skills table above.
3. Commit and open a PR.
