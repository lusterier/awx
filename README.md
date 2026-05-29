# CMDB Software Inventory — AWX Project

Ansible project for collecting software inventory (CMDB) facts from
heterogeneous infrastructure: RHEL/Linux, Windows, AIX, and Oracle Database
hosts. Designed to be selected directly as a playbook in an AWX / Ansible
Automation Platform Job Template.

## Layout

```text
.
├── ansible.cfg
├── requirements.yml
├── cmdb_rhel.yml          # per-platform playbooks at project root
├── cmdb_windows.yml
├── cmdb_aix.yml
├── cmdb_oracle.yml
├── cmdb_all.yml           # imports all of the above
├── roles/
│   ├── rhel_cmdb/
│   ├── win_cmdb/
│   ├── aix_cmdb/
│   └── oracle_cmdb/
└── templates/
    └── cmdb_results.json.j2
```

No `../` paths. Everything required by AWX job isolation is contained inside
this single project tree. `roles_path = ./roles` in `ansible.cfg`.

## Inventory groups

Each playbook targets a group. Define them in your AWX inventory:

| Playbook            | Group     |
|---------------------|-----------|
| `cmdb_rhel.yml`     | `rhel`    |
| `cmdb_windows.yml`  | `windows` |
| `cmdb_aix.yml`      | `aix`     |
| `cmdb_oracle.yml`   | `oracle`  |
| `cmdb_all.yml`      | all of the above |

## Output

Each host appends one or more normalized records to `cmdb_software_results`:

```yaml
cmdb_software_results:
  - hostname: "server01"
    software: "Docker"
    status: "running"          # running | installed | not_installed
    version: "Docker version 26.1.4"
    install_folder: "/usr/bin/docker"
    package: "docker-ce-26.1.4"
    platform: "rhel"           # rhel | windows | aix | oracle
```

Results are published to AWX via `set_stats` with `aggregate: true`, so they
appear as `artifacts.cmdb_software_results` on the Job and flow into the
Workflow data bus.

### Optional JSON artifact

To also write the results to a JSON file inside the job's private data
directory, set `cmdb_write_json: true`. The file is written to
`$AWX_PRIVATE_DATA_DIR/cmdb_software_results.json` (falls back to `/tmp` if
the env var is missing). The project never writes outside the job workspace.

## Per-software collectors

`rhel_cmdb` loads collectors from `roles/rhel_cmdb/tasks/software/<name>.yml`.
Add more by extending `cmdb_rhel_software_modules` in your job extra vars,
e.g. `["docker", "podman", "nginx"]`.

All collectors are read-only:

```yaml
changed_when: false
failed_when: false
```

Playbooks will not fail when a checked piece of software is not installed.

## Syntax check

Run from the project root (the same way AWX runs it):

```bash
ansible-playbook --syntax-check cmdb_rhel.yml
ansible-playbook --syntax-check cmdb_windows.yml
ansible-playbook --syntax-check cmdb_aix.yml
ansible-playbook --syntax-check cmdb_oracle.yml
ansible-playbook --syntax-check cmdb_all.yml
```

## Collections

Install required collections (also resolved automatically by AWX from
`requirements.yml`):

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
```
