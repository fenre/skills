# Skills

Agent skills for Splunk, Cisco, observability, game development, and related workflows. Each skill is a self-contained directory with a `SKILL.md` entry point, optional reference docs, templates, and automation scripts.

## Attribution

A large portion of the Splunk and Cisco skills in this repository are derived from **[splunk-cisco-skills](https://github.com/chambear2809/splunk-cisco-skills)** by [**@chambear2809**](https://github.com/chambear2809), licensed under the [Apache License 2.0](LICENSE).

- **Upstream repository:** https://github.com/chambear2809/splunk-cisco-skills
- **Full attribution inventory:** [ATTRIBUTION.md](ATTRIBUTION.md)
- **License:** [LICENSE](LICENSE)

Imported and overlapping skills include an `ATTRIBUTION.md` file in their directory. Shared scripts and libraries live in `shared/` and are also credited in [shared/ATTRIBUTION.md](shared/ATTRIBUTION.md).

Thank you to @chambear2809 for the original work.

## Repository layout

```
<skill-name>/
├── SKILL.md              # Skill entry point (read this first)
├── reference.md          # Extended reference (when present)
├── template.example      # Non-secret intake worksheet (when present)
├── ATTRIBUTION.md        # Source credit (Splunk/Cisco skills from upstream)
└── scripts/              # Setup, validation, and helper scripts

shared/                   # Shared libraries and registries used by many skills
```

## Using a skill

1. Open the skill's `SKILL.md` for purpose, prerequisites, and workflow.
2. Copy `template.example` to a local file and fill in non-secret values.
3. Run the documented `--help`, dry-run, or preflight command before applying changes.
4. Run validation after setup.

For Cisco product onboarding, start with `cisco-product-setup/SKILL.md` — it resolves a product name and routes to the correct family skill.
