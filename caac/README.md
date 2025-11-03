# Configuration as Code (CaC) for Ansible Automation Platform

This directory contains Ansible playbooks to manage AAP resources using Configuration as Code (CaC) principles.

## Directory Structure

```
caac/
├── controller_vars.yml          # Controller credentials (encrypt with ansible-vault)
├── deploy_main.yaml             # Main playbook for deploying AAP resources
├── files/                        # Place exported inventory files here
│   └── export-results-*.yaml    # Copy exported files from aap-backup
└── roles/
    └── deploy_inventories/
        ├── tasks/
        │   └── main.yml          # Tasks to deploy inventories
        └── vars/
            └── main.yml          # Inventory definitions (fallback if no files)
```

## Usage

### Deploy from Exported Files (Recommended)

1. **Copy exported inventory files** from `aap-backup` to `files/` directory:
   ```bash
   # Pull latest backups from GitHub
   cd ../aap-backup
   git pull
   
   # Copy latest backup file to caac/files
   cp export-results-20251103-131920.yaml ../caac/files/
   ```

2. **Run the deployment playbook**:
   ```bash
   cd ../caac
   ansible-playbook deploy_main.yaml --vault-password-file ~/.vault_pass
   ```

The `deploy_main.yaml` playbook:
- Automatically finds all `export-results-*.yaml` files in `files/` directory
- Loads and transforms the format (flattens `organization.name` → `organization`)
- Uses the `deploy_inventories` role to deploy inventories

### Deploy from Manual Definitions (Fallback)

If no exported files are found in `files/`, the playbook falls back to manual definitions:
- Edit `roles/deploy_inventories/vars/main.yml` with your inventory definitions
- Run: `ansible-playbook deploy_main.yaml --vault-password-file ~/.vault_pass`

## Key Differences Between Export Format and CaC Format

**Export Format (from aap-backup):**
```yaml
inventory:
  - name: "Demo Inventory"
    organization:
      name: "Default"
    related:
      hosts:
        - name: "localhost"
          variables: "..."
      groups: []
```

**CaC Format (for deployment):**
```yaml
controller_inventories:
  - name: "Demo Inventory"
    organization: "Default"
    description: "..."
```

The CaC format is simpler and focuses on declarative definitions, while exports contain full runtime data including hosts, groups, and relationships.

## Implementation Notes

- **Inventory Structure**: Exported inventories include hosts, groups, and variables. The CaC playbook currently only creates inventory containers. Consider extending to include hosts/groups deployment.
- **Organization Mapping**: Ensure organizations exist before deploying inventories.
- **Idempotency**: The ansible.controller.inventory module ensures idempotent operations.

