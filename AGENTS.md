# Repository Guidelines

## Project Structure & Module Organization
This repository is an Ansible playbook for provisioning FreePBX on Ubuntu. The entry point is `cloudpbx-freepbx.yaml`, which applies the roles in order: `sysprep`, `db_prep`, `asterisk`, `freepbx`, `phpmyadmin`, and `monitoring`. Interactive `vars_prompt` asks for the customer name, DB root password and Loki provider label; pass them with `-e` to skip the prompts.

Role logic lives in `roles/<role>/tasks/main.yml`. Role-specific variables are kept in `roles/<role>/vars/main.yml` where present, and Jinja templates live under `roles/<role>/templates/` such as `roles/freepbx/templates/freepbx.service.j2`. Service restarts go through `roles/<role>/handlers/main.yml`. There is currently no `tests/` or `defaults/` tree, so keep new additions consistent with the existing role layout.

## Build, Test, and Development Commands
Use these commands from the repository root:

```sh
export DB_ROOT_PASS='strong-password'
ansible-playbook --syntax-check -i '127.0.0.1,' -u ubuntu cloudpbx-freepbx.yaml
ansible-lint cloudpbx-freepbx.yaml
pre-commit run --all-files
ansible-playbook -i '<server-ip>,' -u ubuntu cloudpbx-freepbx.yaml
```

`--syntax-check` validates playbook structure. `ansible-lint` applies the rules from `.ansible-lint.yaml`. `pre-commit` runs the same lint hook contributors use before committing. A full `ansible-playbook` run is the real deployment path and should be tested on a disposable Ubuntu host.

## Coding Style & Naming Conventions
Write YAML with two-space indentation and start files with `---`. Use fully qualified module names such as `ansible.builtin.apt` and `community.mysql.mysql_user`, matching the existing tasks. Variables must stay lowercase `snake_case`; this is enforced by `var_naming_pattern` in `.ansible-lint.yaml`.

Name tasks with short imperative phrases like `Install phpmyadmin` or `Restart mysql service`. Prefer idempotent tasks with `state`, `creates`, `when`, and templated files ending in `.j2`.

## Testing Guidelines
There is no dedicated automated test suite or Molecule scenario in this repo. At minimum, every change should pass syntax check, `ansible-lint`, and `pre-commit`. For shell-heavy or package-install changes, also verify a clean run on Ubuntu 22.04 and confirm `DB_ROOT_PASS` is supplied via the environment.

## Commit & Pull Request Guidelines
The visible history is minimal and uses short lowercase subjects (`init commit stable ast20/fpbx16`). Keep commit titles brief, specific, and focused on one change. Pull requests should state which roles were touched, what validation was run, whether Ubuntu 20.04 or 22.04 behavior changed, and whether any new external package or download URL was introduced.

## Security & Configuration Tips
Do not hard-code secrets in playbooks, vars files, or inventory. Keep database credentials in `DB_ROOT_PASS`, avoid committing host-specific IPs, and review any change that downloads code or packages from external URLs before merging.
