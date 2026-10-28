# ansible-organization-cac

Team-owned inventory requests for Ansible Automation Platform (AAP) 2.7, built
with configuration as code. AAP configures itself from this repo.

Users in different teams of the same organization (here, **Database Automation**)
request an inventory from a workflow. The organization admin approves the request,
and a job running **as the organization admin** creates the inventory and grants
the selected team **Inventory Admin** on it. Only that team (plus org admins and
system admins) can see the inventory. Other teams can't.

```
 Team member                      Org admin                     Job (runs as org admin)
 ───────────                      ─────────                     ──────────────────────
 Launch "Inventory Request - Workflow (Approval Required)"
   survey: name, description,
           owning team (dropdown)
              │
              ▼
        ┌────────────────────────┐   approve / deny
        │ Administrator Approval │ ◀──────────────
        └────────────────────────┘
              │ approved
              ▼
        ┌─────────────────────────┐
        │ Create Inventory in AAP │  1. check the launching user is in the selected team
        └─────────────────────────┘  2. refuse to take over another team's inventory
                                     3. create the inventory in the organization
                                     4. grant the team "Inventory Admin" on it
```

## Contents

| Path | Purpose |
|---|---|
| `roles/inventory_request` | Runtime role. Creates the inventory and grants the owning team access. |
| `roles/inventory_request_setup` | Configuration as code. Creates the credential, project, job template, workflow, survey, approval node and role assignments in AAP. |
| `playbooks/request_inventory.yml` | Playbook run by the job template inside AAP. |
| `playbooks/setup_workflow.yml` | Playbook you run locally to configure AAP. |
| `collections/requirements.yml` | Red Hat certified collections from Automation Hub (`ansible.platform`, `ansible.controller`). |

## Collections

Both collections are Red Hat certified content from Automation Hub on console.redhat.com.

* `ansible.platform` (2.7.x): looks up teams and team membership in the platform gateway.
* `ansible.controller` (4.8.12 or later): manages inventories, credentials, projects,
  job templates, workflows and role assignments on the controller. `ansible.platform`
  has no modules for these objects.

`ansible.controller` must be **4.8.12 or later** on AAP 2.7 (4.7.18 or later on 2.6).
Earlier certified builds, such as 4.8.0, accept `aap_token` but authenticate only with a
username and password.

The roles work around two issues in the certified `ansible.controller` modules:

* `role_team_assignment` / `role_user_assignment` type `team_ansible_id` /
  `user_ansible_id` as `int`, but ansible_ids are UUIDs. The roles assign by
  controller team id and by username instead.
* `workflow_job_template_node` can't create an `approval_node` and set its
  `success_nodes` in the same call, so the setup does it in two calls.

## Two workflows: an example, not a recommendation

This repo creates two workflows with the same survey, both launchable by every team:

* **Inventory Request - Workflow (Approval Required)**: the org admin approves each request
  before the inventory is created.
* **Inventory Request - Workflow (No Approval Required)**: the inventory is created
  straight away.

> **Note:** Because every team can launch both, anyone can bypass the approval by using the
> No Approval Required workflow. Both are included only to demonstrate the two patterns.
> A real deployment usually picks one, depending on whether approval is needed. Either
> remove the workflow you don't want, or limit who has *WorkflowJobTemplate Execute* on it.
> The team membership check applies in both workflows.

## Security model

* **No secrets in git.** AAP API tokens and the Automation Hub token live in a local,
  git-ignored `secrets.yml`. The org admin's token is stored only in an AAP credential,
  where AAP keeps it encrypted.
* **Token auth only.** No passwords are used anywhere.
* **Least privilege for teams.** Each team gets only *WorkflowJobTemplate Execute* on the
  workflows. Teams get no access to the job template, the project or the org admin credential.
* **Requests are checked server-side.** The survey offers every team, but the job
  confirms through the gateway API that the user who launched the workflow belongs to
  the team they picked. Otherwise the request fails and nothing is created.
* **No takeovers.** Inventory names are unique per organization. A request that matches
  an existing inventory succeeds only if the selected team already owns it.
* **Clean failure.** If granting the team fails, the newly created inventory is deleted
  so no unowned inventory is left behind.
* Survey answers can't override the credential's connection variables, because the
  workflow and job template don't prompt for extra vars.
* **Known trade-off: request history is shared.** Everyone who can launch the shared
  workflow can see its past runs, including the survey answers (inventory name,
  description, team). They can't see or use the inventories themselves. If request
  details must be private too, use one workflow per team instead.

## Prerequisites

* AAP 2.7 with the organization, teams and users already created, and one organization
  admin user.
* The organization's Galaxy credentials must be able to install the certified collections
  in `collections/requirements.yml` during project sync.
* API tokens (*User menu → User details → Tokens*, scope **Write**) for:
  * a user who can configure the organization (a system admin, or the org admin), and
  * the organization admin the jobs run as.
* An Automation Hub offline token from <https://console.redhat.com/ansible/automation-hub/token>.

## Setup

1. Create `secrets.yml` in the repo root. It is git-ignored, so keep it that way.

   ```yaml
   aap_hostname: https://aap.example.com
   aap_validate_certs: true
   aap_token: "<token of the user running setup>"
   org_admin_username: "<org admin username>"
   org_admin_token: "<org admin token>"
   ah_token: "<console.redhat.com offline token>"
   ```

   ```bash
   chmod 600 secrets.yml
   ```

2. Install the certified collections from Automation Hub.

   ```bash
   export ANSIBLE_GALAXY_SERVER_RH_CERTIFIED_TOKEN=$(awk '/^ah_token:/{gsub(/"/,"",$2);print $2}' secrets.yml)
   export ANSIBLE_GALAXY_SERVER_RH_VALIDATED_TOKEN=$ANSIBLE_GALAXY_SERVER_RH_CERTIFIED_TOKEN
   ansible-galaxy collection install -r collections/requirements.yml -p collections
   ```

3. Configure AAP.

   ```bash
   ansible-playbook playbooks/setup_workflow.yml -e @secrets.yml
   ```

   Re-run this playbook any time you change this repo's configuration, add a team, or
   rotate the org admin's token. It is idempotent.

## What the setup creates in AAP

All objects belong to the **Database Automation** organization.

| Object | Name | Who can use it |
|---|---|---|
| Credential (Red Hat Ansible Automation Platform) | Inventory Request - Org Admin API Token | Org admin only |
| Project (this repo) | Inventory Request | Org admin only |
| Inventory (localhost only, runs the job) | Inventory Request - Runner | Org admin only |
| Job template | Inventory Request - Create | Org admin only |
| Workflow job template + survey | Inventory Request - Workflow (Approval Required) | Every team: Execute. Org admin: Approve |
| Workflow job template + survey | Inventory Request - Workflow (No Approval Required) | Every team: Execute |
| Workflow node (approval) | Administrator Approval | Org admin approves or denies (Approval Required workflow only) |
| Workflow node (runs the job template) | Create Inventory in AAP | Runs after approval, or straight away in the No Approval Required workflow |

The setup doesn't store an Automation Hub token. `ah_token` in `secrets.yml` is only for
local `ansible-galaxy` installs.

Re-running the setup re-applies the org admin token from `secrets.yml`. To rotate it,
update `secrets.yml` and run the setup again.

## Using it

1. A team member opens **Templates → Inventory Request - Workflow (Approval Required) → Launch**, fills in the
   inventory name and description, and picks the owning team.
2. The org admin approves the request under **Workflow approvals**.
3. The inventory appears for every member of the selected team, who can manage it
   (Inventory Admin). Members of other teams don't see it.

## Verifying

With one user in team A, one user in team B, and the org admin:

1. User A requests an inventory for team A. The workflow waits for approval.
2. Users A and B can't approve it. The org admin approves.
3. The inventory exists, team A has *Inventory Admin*, and the job output names user A as
   the requester.
4. User A can see and edit the inventory. User B can't see it.
5. User B requests an inventory for team A, and it's approved. The job fails because
   user B isn't in team A, and nothing is created.

## Customizing

Override these role variables, for example in a vars file passed with `-e @myvars.yml`.

| Variable | Default | Description |
|---|---|---|
| `inventory_request_setup_organization` | `Database Automation` | Organization to configure |
| `inventory_request_setup_teams` | `[]` (every team) | Teams in the survey dropdown that get Execute |
| `inventory_request_setup_scm_url` | this repo | Project source |
| `inventory_request_setup_approval_timeout` | `0` | Approval timeout in seconds (0 = never) |
| `inventory_request_setup_execution_environment` | `Default execution environment` | EE for the project and job |
| `inventory_request_organization` | `Database Automation` | Organization the runtime role works in |
| `inventory_request_team_role` | `Inventory Admin` | Role granted to the owning team |

## License

GPL-3.0-or-later
