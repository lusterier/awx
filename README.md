# CMDB Software Inventory — AWX Project

Ansible project for collecting software inventory (CMDB) facts from
heterogeneous infrastructure: **RHEL**, **Oracle Linux**, **Windows**, and
**AIX** hosts. Designed to be selected directly as a playbook in an AWX /
Ansible Automation Platform Job Template.

## Layout

```text
.
├── ansible.cfg                roles_path=./roles, no ../ paths
├── README.md
├── requirements.yml
├── cmdb_rhel.yml              hosts: rhel          → rhel_cmdb
├── cmdb_oracle.yml            hosts: oracle (OL)   → oracle_cmdb
├── cmdb_windows.yml           hosts: windows       → win_cmdb
├── cmdb_aix.yml               hosts: aix           → aix_cmdb
├── cmdb_all.yml               import_playbook of all four
├── roles/
│   ├── rhel_cmdb/             Red Hat Enterprise Linux
│   ├── oracle_cmdb/           Oracle Linux  (the OS — NOT Oracle Database)
│   ├── win_cmdb/              Windows Server
│   └── aix_cmdb/              IBM AIX
└── templates/
    └── cmdb_results.json.j2
```

Each role has the same shape:

```text
roles/<platform>_cmdb/
├── defaults/main.yml          declares cmdb_<platform>_software_modules
└── tasks/
    ├── main.yml               dispatcher: init → loop → optional JSON → set_stats
    └── software/
        └── <name>.yml         one file per software collector (docker, …)
```

No `../` paths anywhere — the whole tree is self-contained so AWX job
isolation can run any playbook directly from this project's root.

## Inventory groups

| Playbook            | Inventory group | Role         |
|---------------------|-----------------|--------------|
| `cmdb_rhel.yml`     | `rhel`          | rhel_cmdb    |
| `cmdb_oracle.yml`   | `oracle`        | oracle_cmdb  |
| `cmdb_windows.yml`  | `windows`       | win_cmdb     |
| `cmdb_aix.yml`      | `aix`           | aix_cmdb     |
| `cmdb_all.yml`      | all of the above |             |

## How to add a new software collector

The role design is built around this loop being trivial. Example: add
**nginx** detection on RHEL hosts.

1. Create `roles/rhel_cmdb/tasks/software/nginx.yml` following the same
   pattern as `docker.yml`:
   - Each probe task is read-only: `changed_when: false`, `failed_when: false`.
   - The final task appends one record to `cmdb_software_results` with the
     normalized schema below.
2. Append `nginx` to `cmdb_rhel_software_modules` in
   `roles/rhel_cmdb/defaults/main.yml`, **or** override at runtime from AWX
   (Extra Vars / Survey):
   ```yaml
   cmdb_rhel_software_modules:
     - docker
     - nginx
   ```
3. Re-run the job. No changes needed in `tasks/main.yml` — the dispatcher
   loop picks up the new module automatically.

The same pattern applies to `oracle_cmdb`, `win_cmdb`, and `aix_cmdb` — each
role owns its own `tasks/software/` tree because detection commands differ
per platform (`rpm` vs Registry vs `lslpp`).

## Normalized result schema

Every collector appends a record with this exact shape:

```yaml
- hostname: "server01"
  software: "Docker"
  status: "running"           # running | installed | not_installed
  version: "Docker version 26.1.4"
  install_folder: "/usr/bin/docker"
  package: "docker-ce-26.1.4"
  platform: "rhel"            # rhel | oracle_linux | windows | aix
```

## Output channels

### `set_stats` (primary, AWX-native)

Each role ends with:

```yaml
- ansible.builtin.set_stats:
    data:
      cmdb_software_results: "{{ cmdb_software_results }}"
    aggregate: true
    per_host: false
```

`aggregate: true` makes Ansible merge each host's list into a single
job-wide list. In AWX this becomes:

- `Job → Details → Artifacts → cmdb_software_results` (downloadable JSON)
- the same key on the Workflow data bus, so downstream nodes can consume it
  without re-querying the inventory.

### Per-host JSON file (optional)

Set `cmdb_write_json: true` (extra var) to also render a per-host JSON
inside the job's writable workspace:

```text
$AWX_PRIVATE_DATA_DIR/cmdb_<inventory_hostname>.json
```

Per-host filename means parallel writes from N hosts do not race. The
project never writes outside `AWX_PRIVATE_DATA_DIR` (falls back to `/tmp`
when running outside AWX).

## Running on many servers — AWX tuning notes

The roles are designed to scale horizontally:

- **Forks** — set the Job Template's *Forks* field (or `--forks N`) to the
  desired concurrency. AWX's defaults (5) are low for inventory scans; 25–50
  is typical for fleet-wide CMDB jobs.
- **Parallel-safe writes** — the optional JSON artifact uses a per-host
  filename (`cmdb_<hostname>.json`), so `delegate_to: localhost` does not
  serialize across hosts.
- **No cross-host barriers** — collectors are pure facts; no
  `wait_for`, `pause`, or `serial: 1` anywhere. Each host runs the full
  dispatcher independently.
- **Read-only probes** — every probe carries `changed_when: false` and
  `failed_when: false`. A host missing the probed software does not fail
  the play and does not mark the host changed (clean idempotent runs).
- **Aggregation server-side** — `set_stats aggregate: true` is computed
  in the controller, so per-host buffers stay small and the final result
  appears once.
- **Inventory targeting** — playbooks use `hosts: <group>`. Use AWX Smart
  Inventories or constructed inventories to drive group membership from
  external sources without touching this repo.
- **Execution Environment** — bundle `requirements.yml` collections
  (`ansible.windows`, `community.general`, `community.windows`) into the
  AWX EE so every run starts ready.

## Syntax check

```bash
ansible-playbook --syntax-check cmdb_rhel.yml
ansible-playbook --syntax-check cmdb_oracle.yml
ansible-playbook --syntax-check cmdb_windows.yml
ansible-playbook --syntax-check cmdb_aix.yml
ansible-playbook --syntax-check cmdb_all.yml
```

Run from the project root, the same way AWX invokes them.

## Collections

Install (mirrors what AWX resolves automatically from `requirements.yml`):

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
```
