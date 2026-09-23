# Changelog

All notable changes to the `acme.windchill` collection. Versions follow
semantic versioning: patch = fix, minor = new role or new optional variable,
major = a variable or role renamed or removed.

## 1.0.0 (2026-09-23)

- Initial release.
- `common`: shared variables, pre-flight checks, working folders, and the
  reusable "LoadFromFile only if the file changed" task file.
- `oir`: object initialization rules rendered from a template (or copied as
  complete load files) and loaded through `common`.
