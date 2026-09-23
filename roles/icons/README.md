# acme.windchill.icons

Copies type icons (and any other images) into Windchill's
`codebase\netmarkets\images`, where the web server serves them from. Soft
types refer to their icon by a path under that folder, so the files must be
on the server **before** the types are imported: `acme.windchill.types`
lists this role as a dependency and Ansible runs it first.

No LoadFromFile is involved. `win_copy` mirrors a folder and only
transfers files whose checksum differs, so the role is idempotent on its
own and `--check --diff` shows exactly which files would change.

## Variables

Set in the configuration repo's `inventory/group_vars/windchill/icons.yml`:

| Variable | Default | Meaning |
|---|---|---|
| `windchill_icons` | `[]` | Folders to mirror (below), in order. |
| `windchill_icons_dest_root` | `<windchill_home>\codebase\netmarkets\images` | Where they go. |

Shared variables come from `acme.windchill.common`.

## One entry = one folder

```yaml
windchill_icons:
  - src: icons                # config/icons/**       ->  netmarkets\images\**
  - src: icons/acme           # config/icons/acme/**  ->  netmarkets\images\acme\**
    dest: acme                # optional subfolder on the server; '' (default) = the root
```

`src` is a folder under `config_dir`; its *contents* are copied, sub-folders
included. Keep the icons in the same sub-folder layout you use in the type
definitions, e.g. a type whose icon is `netmarkets/images/acme/widget.gif`
expects `config/icons/acme/widget.gif` mirrored with `dest: acme` (or the
whole `config/icons` tree mirrored to the root).

## What happens per entry

1. The target folder is created if missing.
2. The folder's files are copied; unchanged files are skipped.

Files that exist on the server but not in `src` are left alone.
