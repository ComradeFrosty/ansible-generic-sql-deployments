# :robot: Repo Structure

## Project Structure Reference

```
ansible-generic-sql-deployments/
├── ansible.cfg                          # Ansible settings — points to inventory
├── site.yml                             # Master playbook — entry point
├── .gitignore                           # Keeps secrets off GitHub
├── README.md
├── inventory/
│   ├── hosts.ini                        # Real credentials — GITIGNORED
│   └── hosts.ini.template               # Placeholder — committed, safe to share
├── group_vars/
│   └── dev_servers.yml                  # Non-secret group variables
└── roles/
    ├── generic-package-installer/       # Deploys packages/binaries
    │   ├── tasks/main.yml
    │   └── defaults/main.yml
    ├── generic-script-runner/           # Runs ordered scripts, halts on error
    │   ├── tasks/main.yml
    │   └── defaults/main.yml
    └── generic-job-executor/            # Period management + job execution loop
        ├── tasks/main.yml
        └── defaults/main.yml
```
