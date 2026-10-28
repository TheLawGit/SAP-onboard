# SAP-onboard

Ansible playbooks for setting up the SAP organization in Ansible Automation Platform (AAP) / AWX and onboarding users to it.

| Playbook | Purpose |
|---|---|
| `setup_sap_controller.yml` | Creates the SAP organization, its teams, project, inventories, test users and the SAP Onboarding job template. |
| `add_user_to_team.yml` | Adds a user to the SAP Executable team and grants the team its access. |
| `create_onboarding_survey.yml` | Adds a survey to the **SAP Onboarding** job template that prompts for the inputs `add_user_to_team.yml` needs. |

Run `setup_sap_controller.yml` first; the other playbooks expect the objects it creates.

## setup_sap_controller.yml

`setup_sap_controller.yml` creates the base objects on the automation controller:

| Object | Name(s) | Details |
|---|---|---|
| Organization | SAP | – |
| Teams | SAP Development, SAP Executable | In SAP |
| Project | sap | Git, `https://github.com/TheLawGit/SAP-onboard.git`, branch `main`, updates from git on every job launch |
| Inventories | SAP NonProd, SAP PROD | In SAP, created empty |
| Users | sap-test-1, sap-test-2 | Members of the SAP organization, password `Ansible123!` |
| Job template | SAP Onboarding | Runs `add_user_to_team.yml` from the `sap` project against SAP NonProd |

Objects that already exist are left as they are, and existing users' passwords are not reset.

## add_user_to_team.yml

`add_user_to_team.yml` runs against the automation controller API and:

1. **Adds a user to the team.** Adds the user given in `username` as a member of the team **SAP Executable** in the **SAP** organization. The team is looked up by name *and* organization, so a same-named team in another organization is never picked.
2. **Lets the team run SAP job templates.** Grants the team the organization-level **Execute** role on **SAP**. Team members can launch every job template (and workflow job template) in that organization, including ones created later. They can't edit them.
3. **Gives the team read access to inventories.** Grants the team the **Read** role on each inventory named in `sap_inventories`. Team members can view hosts, groups and variables but can't change them. This step is skipped if the list is empty.

Every step is idempotent: re-running the playbook changes nothing if the access is already in place.

## create_onboarding_survey.yml

`create_onboarding_survey.yml` enables a survey on the **SAP Onboarding** job template with two questions:

| Question | Type | Required | Variable | Choices |
|---|---|---|---|---|
| **User** | Free-form text | Yes | `username` | – |
| **Inventories** | Multiple select | No | `sap_inventories` | SAP NonProd, SAP PROD |

The survey variables match `add_user_to_team.yml`. When the SAP Onboarding job template runs that playbook, whoever launches it is prompted for the user and can optionally pick inventories. A multiple-select answer is passed as a list. If no inventories are selected, the Read-access step is skipped.

## Requirements

- Ansible with the `ansible.controller` collection (on upstream AWX, use `awx.awx` and swap the module prefix):
  ```bash
  ansible-galaxy collection install ansible.controller
  ```
- A controller account allowed to create organizations, teams, projects, inventories and users, and to manage roles.
- AAP 2.x / AWX 15 or newer.

## Connecting to the controller

Connection details come from environment variables, so no credentials are stored in the playbooks:

```bash
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=...          # or CONTROLLER_OAUTH_TOKEN=...
export CONTROLLER_VERIFY_SSL=false      # optional, for self-signed certificates
```

## Usage

### setup_sap_controller.yml

```bash
ansible-playbook setup_sap_controller.yml
```

To let the SAP Onboarding job reach the controller API, attach an existing **Red Hat Ansible Automation Platform** credential:

```bash
ansible-playbook setup_sap_controller.yml -e "sap_controller_credential=<credential name>"
```

### add_user_to_team.yml

Add a user to the team and grant Execute on SAP:

```bash
ansible-playbook add_user_to_team.yml -e "username=sap-test-1"
```

Also grant Read on specific inventories (lists must be passed as JSON or from a vars file):

```bash
ansible-playbook add_user_to_team.yml -e '{"username": "sap-test-1", "sap_inventories": ["SAP NonProd", "SAP PROD"]}'
```

### create_onboarding_survey.yml

```bash
ansible-playbook create_onboarding_survey.yml
```

## Variables

### setup_sap_controller.yml

| Variable | Default | Description |
|---|---|---|
| `sap_user_password` | `Ansible123!` | Password set on newly created test users. |
| `sap_organization` | `SAP` | Organization to create. |
| `sap_teams` | `SAP Development`, `SAP Executable` | Teams to create in the organization. |
| `sap_project` | `sap` → this repo, branch `main` | Project name, git URL and branch. |
| `sap_inventories_to_create` | `SAP NonProd`, `SAP PROD` | Inventories to create in the organization. |
| `sap_users` | `sap-test-1`, `sap-test-2` | Users to create and add to the organization. |
| `sap_job_template` | `SAP Onboarding` → `add_user_to_team.yml`, inventory `SAP NonProd` | Job template name, playbook and inventory. |
| `sap_controller_credential` | `""` (none) | Controller API credential to attach to the job template. |

### add_user_to_team.yml

| Variable | Default | Description |
|---|---|---|
| `username` | *(required)* | Controller user to add to the team. |
| `sap_inventories` | `[]` | Inventory names the team gets Read on. |
| `team_name` | `SAP Executable` | Team the user is added to. |
| `organization` | `SAP` | Organization the team belongs to, and whose job templates the team can execute. |

### create_onboarding_survey.yml

| Variable | Default | Description |
|---|---|---|
| `job_template_name` | `SAP Onboarding` | Job template the survey is added to. |

## Notes

### setup_sap_controller.yml

- The test users are added to the SAP organization but not to either team; use `add_user_to_team.yml` for that.
- The test user password is stored in plain text in this repo; only use it for test accounts. Existing users' passwords are not changed.
- The SAP Onboarding job template runs `add_user_to_team.yml`, which talks to the controller API. Without `sap_controller_credential` the job will fail at runtime because it has no controller connection details. The execution environment also needs the `ansible.controller` collection.
- The survey is not created here; run `create_onboarding_survey.yml` after this playbook.

### add_user_to_team.yml

- Inventory names are looked up across all organizations. If two organizations have an inventory with the same name, that task fails instead of guessing.
- The playbook fails before making any changes if `username` is missing or the team can't be found in its organization.

### create_onboarding_survey.yml

- The **SAP Onboarding** job template must already exist; the playbook only enables and sets its survey. It's looked up by name only.
- Re-running the playbook replaces the whole survey, overwriting any changes made in the UI.
- The inventory choices must exactly match inventory names in the controller, or the Read grant in `add_user_to_team.yml` will fail.
