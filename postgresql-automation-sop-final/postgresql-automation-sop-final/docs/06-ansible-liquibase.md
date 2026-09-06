# 06 - Ansible + Liquibase

After manual Liquibase validation, status and update work, use Ansible to orchestrate the same workflow.

## Run

```bash
cd ~/ansible-postgresql
ansible-playbook playbooks/liquibase_deploy.yml
```

The playbook:

1. Checks Liquibase.
2. Runs `validate`.
3. Runs `status`.
4. Runs `update`.
5. Verifies `paylite.employee`.

## Idempotency

Run again:

```bash
ansible-playbook playbooks/liquibase_deploy.yml
```

Previously executed changesets should not be executed again.

This is the final manual workflow to hand over to Jenkins.
