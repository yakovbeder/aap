# Organization Management for Ansible Automation Platform

This directory contains Ansible playbooks to manage organizations in AAP using Configuration as Code (CaC) principles. Organizations are created via an AAP Survey, making it easy for authorized users to create organizations without editing code.

## Directory Structure

```
organizations/
├── create_organizations.yml    # Main playbook for creating organizations
├── controller_vars.yml         # Controller connection parameters (secrets-free)
├── survey_spec.yml             # Survey specification reference
└── README.md                   # This file
```

## Security Model

This playbook follows the secure pattern from [Tommer Amber's article](https://medium.com/@tamber/optimizing-cloud-native-operations-series-part-10-how-to-manage-aap-2-5-6e8871e37406):

1. **No secrets in Git**: Credentials are stored in AAP's encrypted credential vault
2. **Machine credential injection**: AAP automatically injects `ansible_user` and `ansible_password`
3. **Temporary tokens**: A one-time-use OAuth2 token is generated for each run
4. **Automatic cleanup**: Token is deleted after playbook completion (even on failure)

## Prerequisites

1. **AAP 2.5 Controller** with admin access
2. **ansible.controller collection** in the Execution Environment (use "Control Plane Execution Environment")

## Setup in AAP

### Step 1: Create the "Controller Admin Login" Credential

1. Navigate to **Automation Execution** → **Infrastructure** → **Credentials**
2. Click **Create credential**
3. Fill in:
   - **Name**: `Controller Admin Login`
   - **Credential Type**: `Machine`
   - **Username**: `admin` (or your admin username)
   - **Password**: Get from Kubernetes secret:
     ```bash
     oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d
     ```
4. Save the credential

### Step 2: Create the Project

1. Navigate to **Automation Execution** → **Projects**
2. Click **Create project**
3. Fill in:
   - **Name**: `Organization Management`
   - **Source Control Type**: `Git`
   - **Source Control URL**: Your Git repository URL
   - **Source Control Branch**: `main`
   - **Options**: Check "Update revision on launch"
4. Save and sync the project

### Step 3: Create the Job Template

1. Navigate to **Automation Execution** → **Templates**
2. Click **Create job template**
3. Fill in:
   - **Name**: `Create Organization`
   - **Inventory**: Select any inventory (e.g., "Demo Inventory")
   - **Project**: `Organization Management`
   - **Playbook**: `organizations/create_organizations.yml`
   - **Execution Environment**: `Control Plane Execution Environment`
   - **Credentials**: Add `Controller Admin Login` (Machine credential)
4. Save the job template

### Step 4: Configure the Survey

1. Edit the Job Template created above
2. Click the **Survey** tab
3. Click **Add question** for each:

#### Question 1: Organization Name (Required)
- **Question**: `Organization Name`
- **Description**: `Name of the organization to create in AAP`
- **Answer variable name**: `org_name`
- **Answer type**: `Text`
- **Required**: Yes
- **Minimum length**: 1
- **Maximum length**: 512

#### Question 2: Organization Description (Optional)
- **Question**: `Organization Description`
- **Description**: `Optional description for the organization`
- **Answer variable name**: `org_description`
- **Answer type**: `Text`
- **Required**: No
- **Maximum length**: 1024

#### Question 3: Maximum Hosts (Optional)
- **Question**: `Maximum Hosts`
- **Description**: `Maximum number of hosts allowed (leave empty for unlimited)`
- **Answer variable name**: `org_max_hosts`
- **Answer type**: `Integer`
- **Required**: No

4. Enable the survey toggle
5. Save the job template

## Usage

### Running from AAP UI

1. Navigate to **Templates**
2. Click **Launch** on the "Create Organization" template
3. Fill in the survey:
   - Enter the organization name
   - Optionally add a description
   - Optionally set max hosts limit
4. Click **Launch**

### Running from CLI (with AAP credentials)

```bash
# Using awx CLI
awx job_templates launch "Create Organization" \
  --extra_vars '{"org_name": "production-org", "org_description": "Production workloads"}'
```

## How Token Generation Works

Based on the [GitOps approach from Tommer Amber](https://medium.com/@tamber/optimizing-cloud-native-operations-series-part-10-how-to-manage-aap-2-5-6e8871e37406):

1. **Credential Injection**: When AAP launches a job with a Machine credential attached, it automatically injects:
   - `ansible_user` (the username)
   - `ansible_password` (the password)

2. **Token Generation**: The playbook uses these credentials to generate a temporary OAuth2 token with `write` scope

3. **Organization Creation**: The token is used for all API calls to create the organization

4. **Cleanup**: The token is deleted using the original username/password (a token cannot delete itself)

```
┌─────────────────────────────────────────────────────────────────┐
│                        AAP Job Template                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Machine Credential: Controller Admin Login                  │ │
│  │   → Injects: ansible_user, ansible_password                 │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Playbook generates temporary OAuth2 token                   │ │
│  │   → Token is short-lived, write-scope only                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Create organization using token                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Delete token (always, even on failure)                      │ │
│  │   → Uses original username/password                         │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Security Best Practices

1. **Protect the Job Template**: Only allow authorized users to execute it
2. **Protect the Credential**: Only allow authorized users to use the Controller Admin Login credential
3. **Protect the Git Repo**: Require pull requests for any changes
4. **Audit**: All changes are logged in AAP job history and Git history

## Files Reference

### controller_vars.yml

```yaml
---
controller_host: "https://aap-controller-aap.apps.yb-ocp4.rh-igc.com"
controller_validate_certs: false
```

This file contains **no secrets**. The credentials are injected by AAP at runtime.

### survey_spec.yml

Reference file documenting the survey configuration. Use this when setting up the survey in AAP UI.

## Troubleshooting

### Error: "Missing required variables"

- Ensure the survey is enabled on the Job Template
- Ensure the user filled in the required "Organization Name" field

### Error: "Invalid authentication credentials"

- Verify the Machine credential has the correct username/password
- Get the password from: `oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 -d`

### Error: "Module ansible.controller.organization not found"

- Ensure you're using the "Control Plane Execution Environment" or a custom EE with `ansible.controller` collection

## Related Resources

- [Tommer Amber's Article: How to Manage AAP 2.5 on OpenShift Like a GitOps Pro](https://medium.com/@tamber/optimizing-cloud-native-operations-series-part-10-how-to-manage-aap-2-5-6e8871e37406)
- [Ansible Controller Collection Documentation](https://console.redhat.com/ansible/automation-hub/repo/published/ansible/controller)
