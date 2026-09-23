# acme.windchill.oir

Loads Windchill **object initialization rules** (OIRs) with
`windchill wt.load.LoadFromFile`, and only reloads a rule when its definition
changed since the last successful load.

The configuration repo's lesson 07 walks through it end to end.

## Variables

Set in the configuration repo's `inventory/group_vars/windchill/oir.yml`:

| Variable | Default | Meaning |
|---|---|---|
| `windchill_oir_rules` | `[]` | List of rules (below). |
| `windchill_oir_force` | `false` | Reload every rule even if unchanged. Handy as `-e windchill_oir_force=true`. |
| `windchill_oir_load_dtd` | `standardX26.dtd` | DTD referenced by generated load files. Must exist in `<WT_HOME>\loadXMLFiles`. |

Shared variables come from `acme.windchill.common` (`windchill_home`,
`windchill_admin_user`, `windchill_admin_password`, `windchill_staging_dir`,
`windchill_default_container_path`, `windchill_load_*`).

## One rule = one dictionary

```yaml
windchill_oir_rules:
  - name: ACME_WTPart                       # required. Rule name in the OIR Administrator
    type: WCTYPE|wt.part.WTPart             # required unless load_file. Type the rule applies to
    container_path: /wt.inf.container.OrgContainer=ACME   # optional, default windchill_default_container_path
    content_file: oir/acme_wtpart.xml       # rule body (<AttributeValues>...), path relative to config_dir
    # content: |                            # ...or the body inline instead of content_file
    #   <AttributeValues objType="wt.part.WTPart"> ... </AttributeValues>
    rule_type: INIT                         # optional, default INIT
    enabled: true                           # optional, default true

  - name: ACME_Change_Rules
    load_file: oir/acme_change_rules.xml    # a complete LoadFromFile document: copied as-is,
                                            # the template is not used
```

`config_dir` is a variable the configuration repo sets (its `config/` folder).

## What happens per rule

1. `tasks/load_rule.yml` renders `templates/oir_load.xml.j2` around the body
   (or copies `load_file`) to `<staging_dir>\staging\oir_<name>.xml` on the server.
2. It hands that file to the shared loader,
   `roles/common/tasks/load_file.yml`, which compares it with the last
   applied copy, runs LoadFromFile only when needed, checks the output,
   saves a log and records the new version as applied.

`--check` renders nothing and loads nothing, but shows which rules would be
(re)loaded.
