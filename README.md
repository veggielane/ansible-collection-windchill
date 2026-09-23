# acme.windchill

Ansible collection that configures PTC Windchill servers on Windows. It is
the **tooling** half of a two-repository setup:

| Repository | Holds | Lifecycle |
|---|---|---|
| this one, `ansible-collection-windchill` | roles (later custom modules); nothing site-specific | tagged, built, published to Artifactory as versions |
| `ansible` (the configuration repo) | inventory, environment values, vaults, the rule bodies in `config/`, playbooks, the lessons | pins a version of this collection and applies it to Windchill |

Nothing in here knows a hostname, a password, a path that belongs to one
server, or an actual Windchill rule. All of that arrives as variables.

## Roles

| Role | Purpose |
|---|---|
| `acme.windchill.common` | Pre-flight checks, working folders, and `tasks/load_file.yml`: the shared "run `wt.load.LoadFromFile` only if the staged file changed" step every other role reuses. |
| `acme.windchill.oir` | Object initialization rules: renders each rule into a load file (or copies a complete one) and hands it to the loader. |

Each role has its own README with the variable table.

## Using it

```yaml
# requirements.yml in the configuration repo
collections:
  - name: acme.windchill
    version: "1.0.0"
```

```yaml
# a playbook
- hosts: "{{ target | default('lab') }}"
  roles:
    - role: acme.windchill.oir
      tags: [oir]
```

Variables the roles expect (`windchill_home`, `windchill_admin_user`,
`windchill_admin_password`, `windchill_staging_dir`, `windchill_oir_rules`,
...) are documented in `roles/common/defaults/main.yml` and
`roles/oir/defaults/main.yml`.

## Developing

Run the checks CI runs:

```bash
ansible-lint --profile production
ansible-playbook tests/selftest.yml                       # loader logic, no Windows host needed
ansible-galaxy collection build --output-path dist
export ANSIBLE_COLLECTIONS_PATH=/tmp/collections     # an empty path, so dependencies are installed too
ansible-galaxy collection install dist/*.tar.gz
ansible-playbook --syntax-check tests/syntax.yml
```

To use an unreleased change from the configuration repo, its
`docker-compose.yml` mounts this checkout as a collection (dev mode). Edits
here are live there; no build or publish needed until you release.

Real behaviour against Windows is exercised from the configuration repo's
lab (a fake Windchill) and its dev server.

## Releasing

1. Update `CHANGELOG.md` and bump `version` in `galaxy.yml`.
2. Merge to `main`, then tag: `git tag v1.1.0 && git push --tags`.
3. The `publish` job builds `acme-windchill-1.1.0.tar.gz` and pushes it to
   Artifactory (`$ARTIFACTORY_URL/api/ansible/$ARTIFACTORY_ANSIBLE_LOCAL_REPO/`).
   It refuses a tag that does not match `galaxy.yml`, and Artifactory
   refuses to overwrite an existing version.
4. In the configuration repo raise the pin in `requirements-windchill.yml`,
   rebuild the control-node image, run the lab, then dev, PPE, live.

Semantic versioning: patch = fix, minor = new role or new optional variable,
major = a variable or role renamed or removed (playbooks or inventories must change).

## CI

`.gitlab-ci.yml` has three stages: `lint` (ansible-lint + a trial build),
`test` (self-test, build, install, syntax check), `publish` (tags only).
The CI/CD variables it needs are listed at the top of that file. Without the
Artifactory variables, lint and test still run using public Galaxy.

## Renaming the namespace

`acme` is a placeholder. To rename it to `yourco`, change:

- `galaxy.yml`: `namespace`
- `roles/oir/meta/main.yml` and `roles/oir/tasks/load_rule.yml`: `acme.windchill.common`
- `tests/syntax.yml` and `.gitlab-ci.yml` (the tarball name in `publish`)
- in the configuration repo: `requirements-windchill.yml`, `docker-compose.yml`
  (the mount path), `playbooks/*.yml`, `playbooks/lab_render_oir.yml`

A project-wide search for `acme.windchill` and `acme/windchill` finds every occurrence.

## Before the first run against a real Windchill

The `oir` role has been exercised against a fake Windchill only. On the dev
server, confirm once:

1. The load-file element names the rule loader accepts for your version:
   `loadFiles\csvmapfile.txt` (element and handler) and
   `loadXMLFiles\standardX*.dtd` (children of `csvTypeBasedRule`). Adjust
   `roles/oir/templates/oir_load.xml.j2` if they differ.
2. The DTD name: `windchill_oir_load_dtd`.
3. A round trip: export an existing rule, load it back unchanged.
4. `windchill_load_failure_patterns` against the output of a successful load.
5. Quoting of `-CONT_PATH` values with spaces (`windchill_load_args`).

## Conventions

- Fully qualified module names everywhere.
- Every variable starts with `windchill_`; role-specific ones with `windchill_<role>_`.
- `defaults/main.yml` documents every variable with a comment and never holds a real value.
- Commands set `changed_when`; anything carrying a password sets `no_log: true`.
- A role that loads files renders to `<windchill_staging_dir>\staging\` and
  includes `acme.windchill.common` with `tasks_from: load_file`.
