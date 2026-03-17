# IntelinAir Skills

Git-managed skill definitions for the IntelinAir Chat Platform.

## How It Works

Skills are YAML plans + XML instructions stored on S3. This repo is the **source of truth** for system/global skills. On push to `main`, a GitHub Actions workflow syncs to S3.

```
Push to main → GitHub Actions → aws s3 sync → S3 bucket (deployed)
```

User-authored skills live only in S3 (no Git needed). System skills are authored here.

## Directory Structure

```
skills/
  {skill-name}/
    main.yaml           ← Execution plan (DAG steps)
    main.xml            ← Agent instructions (interpretation, communication)
    references/         ← Optional sub-plans (for hooks/chaining)
      {ref-name}/
        main.yaml
```

## Skill Anatomy

### main.yaml (Plan)

Defines the execution DAG — steps, parameters, hooks, and reference plan chaining.

```yaml
name: my-skill
version: 1
parameters:
  year: { type: integer, required: true }
steps:
  - step: 1
    tool: query
    output_table: temp_data
    arguments:
      sql: "SELECT * FROM ... WHERE year = {{ year }}"
      table_name: temp_data
  - step: 2
    tool: analyze
    input_tables: [temp_data]
    arguments:
      code: "df = sql_df('SELECT * FROM temp_data')"
  - step: 3
    tool: render
    arguments:
      template: my-dashboard
      source_table: temp_data
hooks:
  - after: 1
    condition: "{{ prescription_types }} contains \"seed\""
    execute: references/seed
```

### main.xml (Instructions)

Tells the agent how to use the skill — what parameters to gather, how to interpret results, what to say to the user.

## Available Skills

| Skill | Description |
|-------|-------------|
| `yp1k` | Yield Per 1,000 seeding rate diagnostic with interactive dashboard |
| `zone-map` | Zone map with variable-rate seeding prescription |
| `zone-prescription` | Multi-type prescription generator (seed, nitrogen, fertilizer) |

## S3 Deployment

Skills deploy to: `s3://intelinair-chat-data/shared/global/skills/all/{skill-name}/`

The GitHub Actions workflow handles sync automatically. Manual deployment:

```bash
aws s3 sync skills/ s3://intelinair-chat-data/shared/global/skills/all/ --delete
```

## Versioning

- **Git**: Full version history, diffs, branches, PRs for system skills
- **S3 Versioning**: Enabled on the bucket — every overwrite preserves the previous version
- **Skill version field**: Each `main.yaml` has a `version:` number for semantic tracking

## Forking Skills

Users can fork any global skill to customize it:

1. Agent reads the global skill: `read(type:"skill", name:"zone-prescription")`
2. User modifies the YAML
3. Agent saves to user scope: `write(type:"skill", name:"my-zone-rx", content:..., instructions:...)`

The user's copy takes precedence over the global version (scope resolution: user > company > global).
