# acme.windchill.common

Shared foundation for every role in the collection. Two parts:

## 1. `tasks/main.yml` - checks, run once per play

Runs automatically before any role that lists `acme.windchill.common` under
`meta/main.yml -> dependencies`. It never changes Windchill itself:

1. Asserts the shared variables are set (`windchill_home`, admin user and
   password, staging dir).
2. Checks `windchill_bin` (normally `bin\windchill.exe`) exists.
3. Creates `<windchill_staging_dir>\staging`, `\applied` and `\logs`.
4. Optionally checks a Windows service is running (`windchill_service_name`).

## 2. `tasks/load_file.yml` - the reusable LoadFromFile step

Most Windchill configuration (OIRs, life cycles, ACL rules, users, folders,
...) is loaded with `windchill wt.load.LoadFromFile`. This task file is the
one implementation of "load this file, but only when it changed":

```yaml
- name: Load the staged file
  ansible.builtin.include_role:
    name: acme.windchill.common
    tasks_from: load_file
  vars:
    windchill_load_file: 'C:\ansible\windchill\staging\oir_ACME.xml'   # already on the server
    windchill_load_label: oir_ACME                                     # names the applied copy and the log
    windchill_load_container_path: /wt.inf.container.OrgContainer=ACME # optional
    windchill_load_force: false                                        # optional
```

| Step | What |
|---|---|
| compare | checksum of the staged file vs `applied\<label>.xml` |
| run | `windchill wt.load.LoadFromFile -d ... -u ... -p ... -CONT_PATH ...` with a timeout |
| log | output saved to `logs\<label>_<timestamp>.log` |
| verify | exit code must be 0 and output must not match `windchill_load_failure_patterns` |
| record | staged file copied to `applied\` so the next run knows it is live |

A role that loads files therefore only has to *produce* the file
(`win_template` / `win_copy`) and include this. See `acme.windchill.oir`.

All variables and their meaning: `defaults/main.yml`.
