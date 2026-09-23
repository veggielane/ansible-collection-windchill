# acme.windchill.types

Imports **soft types** (with their attributes, layouts and global
enumerations) into Windchill with `windchill wt.load.LoadFromFile`, and only
re-imports a file when it changed since the last successful import.

Types must exist before anything that refers to them: an object
initialization rule for `WCTYPE|wt.part.WTPart|com.acme.Widget` fails if the
type has not been created. `acme.windchill.oir` therefore lists this role as
a dependency, so Ansible always runs it first.

## Getting the load files out of Windchill

Type information is moved between Windchill systems as load files, produced
on the source system (typically dev):

- **Type and Attribute Management** (Site ▸ Utilities): export the types
  you want.
- **Export definition file**: write a small XML listing the types, then run
  `windchill wt.load.LoadFromFile -d DefinitionExporter.xml -u <site admin> -p <pw>`
  on the source system. (PTC Help: *Exporting and Importing Type Information*.)

A type's export is **up to four load files**: the type definition, its
attributes, its layouts, and enumerations. Keep one folder per type under
the configuration repo's `config/types/` with those files in it, and point
an entry at the folder. The role loads them in the order given by
`windchill_types_file_order`.

Import semantics, per PTC: new information in the files (types, attributes,
constraints, property values, global enumerations) is added; a type's
layouts are deleted and rebuilt from the file; deletions of subtypes and
attributes that are no longer in the file are written as secondary load
files under `<Windchill>\temp\LWCTypeImpExp` and only take effect if you
load those too. Global enumerations are never deleted.

## Variables

Set in the configuration repo's `inventory/group_vars/windchill/types.yml`:

| Variable | Default | Meaning |
|---|---|---|
| `windchill_types_files` | `[]` | Ordered list of entries (below). |
| `windchill_types_file_order` | `[enum, type, attr, layout]` | Regular expressions matched (case-insensitively) against the file names in a `load_dir`; the first match decides the position, unmatched files go last alphabetically. Adjust to your export's naming. |
| `windchill_types_force` | `false` | Re-import every file even if unchanged. `-e windchill_types_force=true`. |

Shared variables come from `acme.windchill.common`.

## One entry = one dictionary

```yaml
windchill_types_files:
  - name: acme_part                       # required. Names the applied copies and logs
    load_dir: types/acme_part             # a folder under config_dir: every *.xml in it

  - name: acme_document
    files:                                # ...or an explicit list, loaded in this order
      - types/acme_document/AcmeDocument_Enumerations.xml
      - types/acme_document/AcmeDocument_TypeDefinition.xml
      - types/acme_document/AcmeDocument_Attributes.xml
      - types/acme_document/AcmeDocument_Layouts.xml

  - name: site_enumerations
    load_file: types/enumerations.xml     # ...or a single file
    container_path: ''                    # optional on any entry. Empty (default) = site level
```

Entries load in list order; list a parent type's entry before its subtypes'.

## What happens per entry

1. `tasks/resolve_entry.yml` turns the entry into an ordered list of files
   (control-node logic only; `tests/types_order.yml` checks it).
2. For each file, `tasks/load_item.yml` copies it to
   `<staging_dir>\staging\types_<entry>_<file>.xml` and hands it to
   `roles/common/tasks/load_file.yml`, which compares it with the last
   applied copy, runs LoadFromFile only when needed, checks the output,
   saves a log and records the new version as applied.

`--check` copies nothing and loads nothing, but shows which files would be
(re)imported.
