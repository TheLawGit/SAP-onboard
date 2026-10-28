# SAP-onboard

Ansible playbook for onboarding users to the SAP automation team in Ansible Automation Platform (AAP) / AWX.

## What it does

`add_user_to_team.yml` runs against the automation controller API and:

1. **Adds a user to the team.** Adds the user given in `username` as a member of the team **TEAM SAP Executable** in the **SAP Executable** organization. The team is looked up by name *and* organization, so a same-named team in another organization is never picked.
2. **Lets the team run SAP Development job templates.** Grants the team the organization-level **Execute** role on **SAP Development**. Team members can launch every job template (and workflow job template) in that organization, including ones created later. They can't edit them.
3. **Gives the team read access to inventories.** Grants the team the **Read** role on each inventory named in `sap_inventories`. Team members can view hosts, groups and variables but can't change them. This step is skipped if the list is empty.

Every step is idempotent: re-running the playbook changes nothing if the access is already in place.

## Requirements

- Ansible with the `ansible.controller` collection (on upstream AWX, use `awx.awx` and swap the module prefix):
  ```bash
  ansible-galaxy collection install ansible.controller
  ```
- A controller account allowed to manage team membership and roles in the organizations above.
- AAP 2.x / AWX 15 or newer.

## Connecting to the controller

Connection details come from environment variables, so no credentials are stored in the playbook:

```bash
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=...          # or CONTROLLER_OAUTH_TOKEN=...
export CONTROLLER_VERIFY_SSL=false      # optional, for self-signed certificates
```

## Usage

Add a user to the team and grant Execute on SAP Development:

```bash
ansible-playbook add_user_to_team.yml -e "username=jdoe"
```

Also grant Read on specific inventories (lists must be passed as JSON or from a vars file):

```bash
ansible-playbook add_user_to_team.yml -e '{"username": "jdoe", "sap_inventories": ["SAP Prod", "SAP QA"]}'
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `username` | *(required)* | Controller user to add to the team. |
| `sap_inventories` | `[]` | Inventory names the team gets Read on. |
| `team_name` | `TEAM SAP Executable` | Team the user is added to. |
| `organization` | `SAP Executable` | Organization the team belongs to. |
| `job_template_organization` | `SAP Development` | Organization whose job templates the team can execute. |

## Notes

- Inventory names are looked up across all organizations. If two organizations have an inventory with the same name, that task fails instead of guessing.
- The playbook fails before making any changes if `username` is missing or the team can't be found in its organization.
