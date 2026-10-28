---
name: inventory-request-workflow
description: Set up, change, verify, or troubleshoot the team-owned inventory request workflow in this repo (inventory_request and inventory_request_setup roles). Use when adding teams, rotating the org admin token, renaming workflow objects, changing the runtime role, or checking that teams can't see each other's inventories.
---

# Team-owned inventory request workflow

This repo configures AAP (as code) so that team members request an inventory, the
organization admin approves it, and a job running **as the org admin** creates the
inventory and grants **only the selected team** Inventory Admin on it.

## Layout

- `roles/inventory_request`: runtime role, run by the job template after approval.
- `roles/inventory_request_setup`: configuration as code (credential, project, runner
  inventory, job template, workflow + survey, approval node, role assignments).
- `playbooks/setup_workflow.yml`: run locally to configure AAP.
- `playbooks/request_inventory.yml`: run by the job template inside AAP.
- `secrets.yml`: git-ignored. Holds every token and the AAP hostname. Never commit it.

## Common tasks

Run everything from the repo root.

- **Apply configuration changes, add a team, or rotate the org admin token:** update
  `secrets.yml` or the role defaults, then run:
  `ansible-playbook playbooks/setup_workflow.yml -e @secrets.yml`
  The run is idempotent. The survey's team dropdown and the team Execute grants are
  rebuilt from the organization's teams on every run.
- **Rename the workflow or the job template:** rename the existing object in AAP first
  (keeping its ID keeps run history and role assignments), then change the default. If
  you only change the default, the next run creates a second object.
- **Rename workflow nodes:** nodes are matched by `identifier`. A new identifier creates
  a new node, so delete the old nodes before re-running the setup.
- **Install collections locally:** export `ANSIBLE_GALAXY_SERVER_RH_CERTIFIED_TOKEN`
  from `ah_token`, then run
  `ansible-galaxy collection install -r collections/requirements.yml -p collections`.

## Invariants: keep these when changing the roles

1. The runtime role identifies the requester from the **workflow job's `created_by`**
   (`awx_workflow_job_id`), not from the credential the job runs as.
2. It checks **team membership** through `ansible.platform.gateway_api` before creating
   anything. The survey offers every team, so this check is the security boundary.
3. It **refuses to take over** an existing inventory with the same name unless the
   selected team already holds the role on it. Inventory names are unique per org.
4. Creating the inventory and granting the team are in one block with a **rescue** that
   deletes a newly created inventory, so a failure never leaves an unowned inventory.
5. The workflow and job template keep `ask_variables_on_launch: false`, so survey input
   can't override the credential's injected `aap_*` variables.
6. Teams get only *WorkflowJobTemplate Execute*. They get nothing on the job template,
   the project, or the org admin credential.
7. Token-based auth only. `ansible.controller` must be **>= 4.7.18**, because earlier
   builds ignore `aap_token`.

## Verifying isolation

You need API tokens for one user in team A and one in team B, plus the org admin's token.

1. User A launches the workflow for team A. The run waits at the approval node.
2. Users A and B can't approve it (HTTP 403). The org admin approves.
3. The inventory exists, team A holds *Inventory Admin*, and the job output names user A
   as the requester.
4. User A can GET and PATCH the inventory. User B gets count 0 on the list and 403 on PATCH.
5. User B launches the workflow for team A, and it's approved. The job fails with
   "not a member", and nothing is created.
6. As users A and B, fetch the job detail, events, and stdout. All return 403.
7. Delete the test inventories afterwards.

Every user who can launch the workflow can see its run history and survey answers. This
is an accepted trade-off of using a single shared workflow.

## Troubleshooting runtime failures

| Message | Cause |
|---|---|
| `... is not a member of team ...` | Requester picked a team they aren't in (working as intended) |
| `... already exists ... isn't owned by team ...` | The name is already used by another team's inventory |
| `Could not work out which user launched this request` | Job launched outside the workflow without `awx_user_name` |
| `Invalid authentication credentials ... (HTTP 401)` | Org admin token expired or revoked, or `ansible.controller` < 4.7.18. Rotate the token and re-run the setup |
| Project sync fails installing collections | The organization's Galaxy credentials can't reach the certified collections |
