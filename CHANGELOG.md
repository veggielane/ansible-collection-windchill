# Changelog

All notable changes to the `acme.windchill` collection. Versions follow
semantic versioning: patch = fix, minor = new role or new optional variable,
major = a variable or role renamed or removed.

## 1.5.0 (2026-09-23)

- `properties`: the plan compares the current override's target file as
  well as its value, so changing an entry's `target` moves the property.
  New `windchill_properties_force` re-sets every present property.

## 1.4.0 (2026-09-23)

- New role `properties`: manages Windchill properties as `site.xconf`
  overrides through `xconfmanager` (read current override, set only what
  differs, `--reset` for `state: absent`, propagate once), then restarts
  Windchill via `windchill_restart_command` or reports that a restart is
  needed. Handlers are flushed at the end of the role so later roles see
  the new values. `tests/properties_plan.yml` checks the diff logic.

## 1.3.0 (2026-09-23)

- New role `icons`: mirrors folders of images from the configuration repo
  into `codebase\netmarkets\images` with `win_copy` (idempotent by
  checksum, no LoadFromFile).
- `common`: the loader now joins LoadFromFile's stdout and stderr with a real
  newline (it was a literal backslash-n).
- `types` now depends on `icons` (after `common`): a type's icon file is on
  the server before the type definition that refers to it is imported. The
  chain is `oir -> types -> icons -> common`.

## 1.2.0 (2026-09-23)

- `types`: an entry can now be a folder (`load_dir`) holding one type's
  export, which is up to four load files (type definition, attributes,
  layouts, enumerations). They are loaded in the order given by the new
  `windchill_types_file_order` (default: enumerations, type, attributes,
  layouts); explicit `files:` lists and single `load_file` entries remain.
- `tests/types_order.yml` checks the ordering logic on the control node.

## 1.1.0 (2026-09-23)

- New role `types`: imports soft types, attributes, layouts and global
  enumerations from load files exported out of Type and Attribute
  Management, through the shared loader. Site level by default.
- `oir` now depends on `types` (after `common`), so type definitions are
  always imported before the rules that refer to them.

## 1.0.0 (2026-09-23)

- Initial release.
- `common`: shared variables, pre-flight checks, working folders, and the
  reusable "LoadFromFile only if the file changed" task file.
- `oir`: object initialization rules rendered from a template (or copied as
  complete load files) and loaded through `common`.
