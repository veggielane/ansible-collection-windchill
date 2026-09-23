# acme.windchill.types

Imports **soft types** (with their attributes, layouts and global
enumerations) into Windchill with `windchill wt.load.LoadFromFile`, and only
re-imports a file when it changed since the last successful import.

Types must exist before anything that refers to them: an object
initialization rule for `WCTYPE|wt.part.WTPart|com.acme.Widget` fails if the
type has not been created. `acme.windchill.oir` therefore lists this role as
a dependency, so Ansible always runs it first.

## Getting the load file out of Windchill

Type information is moved between Windchill systems as a load file, produced
on the source system (typically dev) in one of two ways:

- **Type and Attribute Management** (Site ▸ Utilities): export the types
  you want; the download contains the load-file XML.
- **Export definition file**: write a small XML listing the types, then run
  `windchill wt.load.LoadFromFile -d DefinitionExporter.xml -u <site admin> -p <pw>`
  on the source system, which writes the export load file(s). (PTC Help:
  *Exporting and Importing Type Information*.)

Save the resulting XML under the configuration repo's `config/types/` and
list it in `windchill_types_files`.

Import semantics, per PTC: the target's information for each type in the
file is **replaced**; anything for that type that is not in the file is
deleted. Global enumerations are never deleted. Deletions of subtypes and
attributes are written as a secondary load file under
`<Windchill>\temp\LWCTypeImpExp` and only take effect if you load that too.

## Variables

Set in the configuration repo's `inventory/group_vars/windchill/types.yml`:

| Variable | Default | Meaning |
|---|---|---|
| `windchill_types_files` | `[]` | Ordered list of exported load files (below). |
| `windchill_types_force` | `false` | Re-import every file even if unchanged. `-e windchill_types_force=true`. |

Shared variables come from `acme.windchill.common`.

## One file = one dictionary

```yaml
windchill_types_files:
  - name: acme_part_types                   # required. Names the applied copy and the log
    load_file: types/acme_part_types.xml    # required. Relative to config_dir; copied as-is
    container_path: ''                      # optional. Empty (default) = no -CONT_PATH = site level
  - name: acme_document_types
    load_file: types/acme_document_types.xml
```

Order matters when one file's types extend another's: list parents first.

## What happens per file

1. `tasks/load_item.yml` copies the file to `<staging_dir>\staging\types_<name>.xml`.
2. It hands that file to `roles/common/tasks/load_file.yml`, which compares
   it with the last applied copy, runs LoadFromFile only when needed, checks
   the output, saves a log and records the new version as applied.

`--check` copies nothing and loads nothing, but shows which files would be
(re)imported.
