# Configuration as Code (CaC) for Ansible Automation Platform

This directory contains Ansible playbooks to manage AAP resources using Configuration as Code (CaC) principles. The playbook can deploy inventories from exported backup files or manual definitions.

## Directory Structure

```
caac/
├── controller_vars.yml          # Controller credentials (MUST be encrypted with ansible-vault)
├── deploy_main.yaml             # Main playbook for deploying AAP resources
├── files/                        # Place exported inventory files here
│   └── .gitkeep                 # Keep this directory in git
├── README.md                    # This file
└── roles/
    └── deploy_inventories/
        ├── tasks/
        │   └── main.yml         # Tasks to deploy inventories
        └── vars/
            └── main.yml         # Manual inventory definitions (fallback)
```

## Prerequisites

1. **Ansible installed** (version 2.9+) OR **AAP 2.5 Controller** access
2. **ansible.controller collection** installed (if running locally):
   ```bash
   ansible-galaxy collection install ansible.controller
   ```
3. **Access to AAP Controller** with admin credentials
4. **Vault password file** (for local execution) OR **Vault credential in AAP 2.5** (for execution from AAP)

## Initial Setup

### Step 1: Create Vault Password File (Recommended)

Create a secure file to store your vault password:

```bash
# Create vault password file (choose a secure location)
echo "your-secure-vault-password" > ~/.vault_pass

# Set secure permissions
chmod 600 ~/.vault_pass
```

**Important:** Never commit the vault password file to git! Add it to `.gitignore`.

### Step 2: Configure Controller Credentials

Edit `controller_vars.yml` with your AAP controller details:

```bash
# Edit the controller variables file
vi controller_vars.yml
```

Update these values:
- `controller_host`: Your AAP controller URL
- `controller_username`: Admin username
- `controller_password`: Admin password

Example:
```yaml
---
controller_host: "https://aap-controller-aap.apps.yb-ocp4.rh-igc.com"
controller_validate_certs: false
controller_username: "admin"
controller_password: "your-password-here"
```

### Step 3: Encrypt controller_vars.yml with Ansible Vault

**CRITICAL:** This file contains sensitive credentials and MUST be encrypted before committing to git.

```bash
# Encrypt controller_vars.yml using vault password file
ansible-vault encrypt controller_vars.yml --vault-password-file ~/.vault_pass

# Alternative: Encrypt and enter password interactively
ansible-vault encrypt controller_vars.yml
```

**Verify encryption:**
```bash
# This should show encrypted content
cat controller_vars.yml
```

You should see `$ANSIBLE_VAULT;1.1;AES256` at the beginning of the file.

## Running with AAP 2.5 (Job Template Execution)

If you plan to run this playbook from AAP 2.5 as a Job Template, you need to create a Vault credential in AAP to decrypt `controller_vars.yml`.

### Step 1: Create Vault Credential in AAP 2.5

1. **Navigate to Credentials in AAP Web UI:**
   - Go to **Resources** → **Credentials**
   - Click **Add** → **Add Credential**

2. **Configure Vault Credential:**
   - **Name:** `CaC Vault Password` (or your preferred name)
   - **Credential Type:** `Ansible Vault`
   - **Vault Password:** Enter your vault password (the same one used to encrypt `controller_vars.yml`)
   - **Organization:** Select appropriate organization (e.g., "Default")
   - **Description:** `Vault password for decrypting controller_vars.yml in CaC playbook`

3. **Save the credential**

### Step 2: Create Project in AAP

1. **Navigate to Projects in AAP Web UI:**
   - Go to **Resources** → **Projects**
   - Click **Add** → **Add Project**

2. **Configure Project:**
   - **Name:** `CaC Configuration` (or your preferred name)
   - **Organization:** Select appropriate organization
   - **SCM Type:** `Git`
   - **SCM URL:** URL to your git repository containing this caac folder
   - **SCM Branch/Tag/Commit:** `main` (or your branch)
   - **SCM Update Options:** Check `Clean` and `Update Revision on Launch`

3. **Save and Sync the project**

### Step 3: Create Job Template in AAP

1. **Navigate to Job Templates in AAP Web UI:**
   - Go to **Resources** → **Templates**
   - Click **Add** → **Add Job Template**

2. **Configure Job Template:**
   - **Name:** `Deploy CaC Inventories` (or your preferred name)
   - **Job Type:** `Run`
   - **Inventory:** Select an inventory (can be a minimal/localhost inventory)
   - **Project:** Select the project created in Step 2
   - **Playbook:** `caac/deploy_main.yaml` (adjust path based on your repository structure)
   - **Credentials:** 
     - Add the **Vault credential** created in Step 1
     - Optionally add **Machine credential** for localhost if needed
   - **Verbosity:** `Normal` (or `Verbose` for debugging)
   - **Extra Variables:** (optional)
     ```yaml
     # If you need to override controller connection variables
     controller_host: "https://aap-controller-aap.apps.yb-ocp4.rh-igc.com"
     controller_validate_certs: false
     ```
   - **Options:** 
     - Check `Enable Privilege Escalation` if needed
     - Check `Use Fact Cache` if you want to cache facts

3. **Save the job template**

### Step 4: Run Job Template

1. Click **Launch** on the job template
2. The playbook will:
   - Automatically decrypt `controller_vars.yml` using the Vault credential
   - Load exported files from `files/` directory (if present in repository)
   - Deploy inventories to AAP

### Important Notes for AAP 2.5 Execution

- **Repository Structure:** Ensure the `caac/` folder is accessible in your AAP project
- **Files Directory:** The exported inventory files should be committed to the repository in `caac/files/` or synced via the project
- **Vault Credential:** Must match the password used to encrypt `controller_vars.yml`
- **Execution Environment:** Ensure the execution environment has the `ansible.controller` collection installed

## Usage (Local Execution)

If running the playbook locally (not from AAP), use these instructions.

### Option 1: Deploy from Exported Backup Files (Recommended)

1. **Copy exported inventory files** from `aap-backup` to `files/` directory:

   ```bash
   # Navigate to aap-backup directory and pull latest
   cd ../aap-backup
   git pull
   
   # Copy latest backup file to caac/files
   cp export-results-20251103-131920.yaml ../caac/files/
   
   # Or copy all backup files
   cp export-results-*.yaml ../caac/files/
   
   # Navigate back to caac
   cd ../caac
   ```

2. **Run the deployment playbook**:

   ```bash
   # Using vault password file (recommended)
   ansible-playbook deploy_main.yaml --vault-password-file ~/.vault_pass
   
   # Or enter vault password interactively
   ansible-playbook deploy_main.yaml --ask-vault-pass
   ```

The playbook will:
- Automatically find all `export-results-*.yaml` files in `files/`
- Transform the format (flattens `organization.name` → `organization`)
- Deploy inventories using the `deploy_inventories` role

### Option 2: Deploy from Manual Definitions

If no exported files are found in `files/`, the playbook automatically falls back to manual definitions:

1. **Edit manual inventory definitions**:

   ```bash
   vi roles/deploy_inventories/vars/main.yml
   ```

   Example:
   ```yaml
   ---
   controller_inventories:
     - name: "Demo Inventory"
       organization: "Default"
       description: "A sample inventory for demonstrations"
   ```

2. **Run the deployment playbook**:

   ```bash
   ansible-playbook deploy_main.yaml --vault-password-file ~/.vault_pass
   ```

## Vault Credential Management in AAP 2.5

### Viewing Vault Credential in AAP

The vault password is stored encrypted in AAP. You cannot view it directly, but you can:
- Edit the credential to change the password
- Delete and recreate if needed

### Updating Vault Password

If you change the vault password:

1. **Re-encrypt controller_vars.yml locally:**
   ```bash
   # Decrypt with old password
   ansible-vault decrypt controller_vars.yml
   
   # Re-encrypt with new password
   ansible-vault encrypt controller_vars.yml
   
   # Commit the updated encrypted file
   git add controller_vars.yml
   git commit -m "Update vault encryption"
   ```

2. **Update Vault Credential in AAP:**
   - Go to **Resources** → **Credentials**
   - Find your Vault credential
   - Click **Edit**
   - Update the **Vault Password** field
   - **Save**

### Troubleshooting Vault in AAP

**Error:** `Vault password not provided`
- Solution: Ensure the Vault credential is added to the Job Template credentials list

**Error:** `Decryption failed`
- Solution: Verify the vault password in the AAP credential matches the one used to encrypt the file

**Error:** `Could not find vault file`
- Solution: Ensure `controller_vars.yml` is in the project path specified in the Job Template

## Common Operations (Local Execution)

### View Encrypted File (without editing)

```bash
# View encrypted controller_vars.yml
ansible-vault view controller_vars.yml --vault-password-file ~/.vault_pass

# Or interactively
ansible-vault view controller_vars.yml
```

### Edit Encrypted File

```bash
# Edit encrypted controller_vars.yml
ansible-vault edit controller_vars.yml --vault-password-file ~/.vault_pass

# Or interactively
ansible-vault edit controller_vars.yml
```

### Re-encrypt After Changes

If you need to re-encrypt after making changes:

```bash
ansible-vault encrypt controller_vars.yml --vault-password-file ~/.vault_pass
```

### Change Vault Password

```bash
# Rekey the vault file with a new password
ansible-vault rekey controller_vars.yml --vault-password-file ~/.vault_pass
```

### Decrypt File (for backup or migration)

```bash
# Decrypt controller_vars.yml (use carefully!)
ansible-vault decrypt controller_vars.yml --vault-password-file ~/.vault_pass
```

**Warning:** After decrypting, remember to encrypt again before committing to git!

### Check What Will Be Deployed

The playbook displays what it will deploy before running:

```bash
ansible-playbook deploy_main.yaml --vault-password-file ~/.vault_pass
```

Look for the debug output:
```
Source: Exported files
Will deploy 3 inventories: ['Demo Inventory', 'yb-hosts-static', 'yb-vmware-dynamic']
```

or

```
Source: Manual vars (roles/deploy_inventories/vars/main.yml)
Will deploy 2 inventories: ['Demo Inventory', 'Production Web Servers']
```

## Workflow Example

Complete workflow for syncing from backups:

```bash
# 1. Navigate to caac directory
cd /Users/ybeder/Documents/OCP/aap/caac

# 2. Pull latest backups from aap-backup repository
cd ../aap-backup
git pull
cd ../caac

# 3. Copy latest backup file
LATEST=$(ls -t ../aap-backup/export-results-*.yaml | head -1)
cp "$LATEST" files/

# 4. Deploy inventories from backup
ansible-playbook deploy_main.yaml --vault-password-file ~/.vault_pass

# 5. Verify deployment was successful
# Check the playbook output for "changed" or "ok" status
```

## Troubleshooting

### Vault Password Issues

**Error:** `Vault password file not found`
```bash
# Solution: Ensure vault password file exists and has correct permissions
ls -la ~/.vault_pass
chmod 600 ~/.vault_pass
```

**Error:** `Decryption failed`
```bash
# Solution: Check if you're using the correct vault password
# Try viewing the file
ansible-vault view controller_vars.yml --vault-password-file ~/.vault_pass
```

### File Not Found Errors

**Error:** `Could not find or access 'files/' directory`
```bash
# Solution: Ensure files directory exists
mkdir -p files
touch files/.gitkeep
```

### Collection Not Found

**Error:** `couldn't resolve module/action 'ansible.controller.token'`
```bash
# Solution: Install the ansible.controller collection
ansible-galaxy collection install ansible.controller
```

### No Inventories to Deploy

**Error:** `No inventories found`
```bash
# Check if files exist in files/ directory
ls -la files/

# Check if manual vars are defined
cat roles/deploy_inventories/vars/main.yml
```

## Security Best Practices

1. **Always encrypt `controller_vars.yml`** before committing to git
2. **Never commit vault password files** - add to `.gitignore`:
   ```bash
   echo ".vault_pass" >> .gitignore
   echo "*_pass" >> .gitignore
   ```
3. **Use vault password files** instead of `--ask-vault-pass` in scripts
4. **Set secure permissions** on vault password file:
   ```bash
   chmod 600 ~/.vault_pass
   ```
5. **Rotate vault passwords** periodically:
   ```bash
   ansible-vault rekey controller_vars.yml --vault-password-file ~/.vault_pass
   ```

## Format Transformation

The playbook automatically transforms exported inventory format:

**Export Format (from aap-backup):**
```yaml
inventory:
  - name: "Demo Inventory"
    organization:
      name: "Default"  # Nested structure
```

**CaC Format (for deployment):**
```yaml
controller_inventories:
  - name: "Demo Inventory"
    organization: "Default"  # Flat structure
```

Only the `organization.name` field is flattened - all other fields match directly.

## Support

For issues:
1. Check playbook output for error messages
2. Verify controller credentials are correct
3. Ensure exported files are in the correct format
4. Verify ansible.controller collection is installed
