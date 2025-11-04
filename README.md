# ansible-role-vault-promote

Promote Vault (or other secrets provider) lookups into Ansible facts so that playbooks
don't repeatedly perform slow lookups against a third-party secret provider.

This role centralizes secret fetching and exposes secrets as facts with a configurable
prefix. Use it when you want to perform Vault lookups once per host (or once per play)
and then reuse the values quickly across later tasks, templates or handlers.

## Key goals

- Cache secrets retrieved from Vault/secret backend in Ansible facts.
- Avoid repeated slow lookup() calls during a play run.

## Requirements

- Ansible 2.9+ (or later, depending on your project standards).
- Network access and authentication to your secret provider (HashiCorp Vault, Vaultwarden, etc.).

This role does not attempt to manage Vault server lifecycle; it only performs lookups
and stores results as facts for the current play host.

## Role variables

Below are common variables you can set when using this role. They are written as
examples — adjust the names/values to match your inventory and secrets layout.

- `vault_promote_var_to_extract` (dict, default: {})
	- Allow you to declare a set of variable that should be unpacked and set as fact. 

- `vault_promote_no_log` (bool, default: true)
	- If `true`, hide ansible output (to protect secrets) `false`, show ansible output (should be only use for debugging purpose).


## Example usage

Simple example using a list of secrets (per-host):

```yaml
- hosts: appservers
    vars:
        nginx_certbot_appname:
          nginx_certbot_domain_name: 'some.url'
          nginx_certbot_port: 8080        
        vault_promote_var_to_extract:
        nginx_certbot: "nginx_certbot_appname"
        nginx_certbot_app: "nginx_certbot_appname_app"

    roles:
        - chadek.vault_promote
```

After the role runs, you can access 
directly `nginx_certbot_domain_name` instead of `nginx_certbot_appname.nginx_certbot_domain_name`

`vault_host` and `vault_group` variables will be evaluated and directly accessible as fact. In order not to override `vault_group` content across groups of the inventory, you should use `vault_group_name_group` in every group of your inventory to wrapp your group secret so they'll be merged into `vault_group` variable by the role itself.

Example:
```yaml
vault_test_group:
  test_admin_passwd: "{{ lookup('community.general.bitwarden', 'vault_test_admin_passwd', collection_name='ansible-secret/production/group_vars/test', field='password')[0] }}"
  test_db_passwd: "{{ lookup('community.general.bitwarden', 'vault_test_db_passwd', collection_name='ansible-secret/production/group_vars/test', field='password')[0] }}"
```

## Behavior and idempotence

- The role performs lookups and writes results into `ansible_facts` for the duration
	of the current play. Facts are not persisted between separate `ansible-playbook` runs
	unless you explicitly register or store them (for example, in a file or vault).
- Running the role multiple times during a single play is safe — the latest fetched
	values will overwrite the facts.

## Security notes

- Secrets fetched by this role are available as in-memory Ansible facts during the
	run. Avoid printing them in logs or verbose output. Use `no_log: true` on tasks
	that might expose secrets.
- Prefer ephemeral tokens or AppRole where possible instead of long-lived tokens.

## Files and structure

Look in the role for the canonical implementation details:

- `defaults/main.yml` — default variable names and sensible defaults.
- `tasks/main.yml` — the lookup logic and any conditional behavior.
- `handlers/main.yml` — (if present) handlers that the role triggers.

## Author and license

This role is provided as part of the repository playbooks. Check the repository
LICENSE file for licensing information.

## Notes

If you'd like, I can update the README later to reflect the exact variable names
and examples used by this repository's implementation — tell me if you want me to
parse `defaults/main.yml` and `tasks/main.yml` and generate a verbatim mapping.
