# pm-preset-bug-triage

Strict bug triage preset for [pm-cli](https://github.com/unbraind/pm-cli) workspaces.

Configures your workspace for production incident tracking with mandatory root-cause and resolution metadata. All governance rules are set to `strict` — items **cannot be closed** without complete metadata.

---

## Install

```bash
pm install github.com/unbraind/pm-preset-bug-triage
```

The `--project` flag scopes the extension to the current workspace only.

---

## Usage

From the root of your project (where `.agents/pm/` lives):

```bash
pm triage-setup
```

This writes `.agents/pm/settings.json` and installs three item templates under `.agents/pm/templates/`.

---

## Options

| Flag | Type | Default | Description |
|---|---|---|---|
| `--force` | boolean | `false` | Overwrite an existing `settings.json` without prompting |
| `--dry-run` | boolean | `false` | Preview all writes without touching the filesystem |
| `--prefix <str>` | string | `bug-` | Override the `id_prefix` applied to all item IDs |

Examples:

```bash
# Preview what would be written
pm triage-setup --dry-run

# Force-overwrite existing settings
pm triage-setup --force

# Use a custom ID prefix
pm triage-setup --prefix inc-
```

---

## What gets applied

### Settings (`settings.json`)

| Setting | Value |
|---|---|
| `id_prefix` | `bug-` |
| `governance.preset` | `strict` |
| `governance.ownership_enforcement` | `strict` |
| `governance.create_mode_default` | `strict` |
| `governance.close_validation_default` | `strict` |
| `governance.metadata_profile` | `strict` |
| `validation.sprint_release_format` | `strict_error` |
| `validation.parent_reference` | `warn` |
| `testing.record_results_to_items` | `true` |
| `search.mode` | `keyword` |
| `telemetry.enabled` | `false` |

### Item types

| Name | Description |
|---|---|
| `Issue` | A defect, incident, or regression requiring investigation and resolution |
| `Task` | A remediation, hotfix, or follow-up task linked to an incident |

---

## Templates

### `incident` — Production Incident (priority: critical)

For active production incidents. Requires:

- `severity` (default: `sev2`)
- `environment` (default: `production`)
- `detected_at`, `reported_by`, `owner`
- `affected_systems`, `affected_users`
- `steps_to_reproduce`
- **`root_cause`** — required to close
- **`resolution`** — required to close
- `mitigation_applied`, `postmortem_url`, `linked_hotfix`

```bash
pm create --template incident
```

### `hotfix-task` — Hotfix Task (priority: critical)

For the engineering work to fix an incident. Links back to a parent incident via `linked_incident`. Requires:

- `linked_incident`, `assignee`
- `fix_description`, `pr_link`
- `target_branch` (default: `main`), `deploy_target` (default: `production`)
- `rollback_plan`, `reviewed_by`
- `deployed_at`, `verified_by`

```bash
pm create --template hotfix-task
```

### `regression` — Regression (priority: high)

For regressions discovered in production or QA. Requires:

- `severity` (default: `sev3`)
- `environment`, `introduced_in`, `last_known_good_version`
- `steps_to_reproduce`, `expected_behavior`, `actual_behavior`
- `owner`, `affected_tests`
- **`root_cause`** — required to close
- `fix_pr`, `verified_fixed_in`

```bash
pm create --template regression
```

---

## Strict governance implications

This preset sets all governance rules to `strict`. Concretely:

- **Creating items** requires all mandatory fields to be provided upfront (`create_mode_default: strict`).
- **Closing items** is blocked until `root_cause` and `resolution` fields are populated (`close_validation_default: strict`, `metadata_profile: strict`).
- **Ownership** must be assigned on every item — unowned items cannot be created or transitioned (`ownership_enforcement: strict`).
- **Sprint/release references** that do not match the expected format are treated as hard errors, not warnings (`sprint_release_format: strict_error`).

This is intentional. Bug triage without a root cause is not triage — it's a list of complaints.

---

## Incident response workflow

A typical flow using this preset:

```
1. Production alert fires
   pm create --template incident
   # Fill in: severity, affected_systems, owner, detected_at

2. Engineer picks up the incident
   pm update bug-001 --meta owner=alice

3. Mitigation deployed
   pm update bug-001 --meta mitigation_applied="Rolled back deploy #482"

4. Hotfix in progress
   pm create --template hotfix-task
   # Fill in: linked_incident=bug-001, assignee=alice, pr_link=...

5. Hotfix merged and deployed
   pm update bug-002 --meta deployed_at="2026-05-09T14:32Z" verified_by=bob

6. Incident resolved — root cause documented
   pm update bug-001 --meta root_cause="OOM in payment-service due to unbounded cache" \
                          resolution="Deployed hotfix #483, cache TTL capped at 60s"
   pm close bug-001

7. Postmortem written
   pm update bug-001 --meta postmortem_url="https://notion.so/postmortem-bug-001"
```

---

## License

MIT

## Release Automation

This package is release-ready for GitHub, npm, and Bun-compatible installs. CI runs type checking, build, production dependency audit, package packing, Bun install verification, and pm-changelog validation. The daily release workflow publishes only when commits exist after the latest release tag and uses pm-changelog to generate CHANGELOG.md and GitHub release notes.
