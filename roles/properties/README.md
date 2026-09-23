# acme.windchill.properties

Manages Windchill properties (`wt.properties`, `db.properties`, ...) the
supported way: as site overrides in `site.xconf`, set with `xconfmanager`
and propagated into the property files. Different pattern from the
loaders: **read the current value, change only what differs, then restart**.

## How it works

| Step | What |
|---|---|
| read | PowerShell parses `site.xconf` and returns the current override of every wanted property |
| plan | control-node logic decides which ones differ (`tests/properties_plan.yml` checks it) |
| apply | `xconfmanager -s name=value ... -t <target>` once per target file; `xconfmanager --reset name` for `state: absent`; then `xconfmanager -p` |
| restart | the `windchill properties changed` handler runs `windchill_restart_command` (or just says a restart is needed), flushed at the end of the role so later roles see the new values |

Re-running with nothing changed makes no `xconfmanager` call. `--check`
shows the plan and calls nothing.

## Variables

Set in the configuration repo's `inventory/group_vars/windchill/properties.yml`
(and the restart command in `vars.yml`):

| Variable | Default | Meaning |
|---|---|---|
| `windchill_properties` | `[]` | The properties to enforce (below). |
| `windchill_properties_default_target` | `codebase/wt.properties` | Property file used when an entry has no `target`. Set `target` for properties that belong elsewhere (`codebase/db/db.properties`, ...). |
| `windchill_xconfmanager` | `<windchill_home>\bin\xconfmanager.exe` | The utility. |
| `windchill_site_xconf` | `<windchill_home>\site.xconf` | Where overrides are recorded. |
| `windchill_restart_command` | `''` | PowerShell that restarts your Windchill. Empty = only report that a restart is needed. |
| `windchill_properties_failure_patterns` | `[exception, \berror\b]` | Output that means xconfmanager failed despite exit code 0. |

## One entry = one dictionary

```yaml
windchill_properties:
  - name: wt.mail.from
    value: windchill@example.com
  - name: wt.method.verboseClient
    value: 'false'                          # quote booleans and numbers to keep them verbatim
  - name: wt.pom.queryLimit
    value: '2000'
    target: codebase/db/db.properties       # not wt.properties: say so
  - name: some.obsolete.property
    state: absent                           # drop the site override (--reset to the declared default)
```

An unquoted YAML `true` or `30` is normalised to `true` / `30` anyway;
quoting just makes the intent obvious.

## Notes

- Only *overrides* are compared. A property that is not in `site.xconf` yet
  is set on the first run even if its declared default already has that
  value; from then on it is stable.
- `xconfmanager -d <name>` on the server shows a property's current value
  and where it comes from, handy when checking what the role did.
- The restart happens inside this role (handlers are flushed at its end),
  so put the role first in `site.yml` when later roles need the new values.
