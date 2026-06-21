# homelab

Ansible-managed bare-metal k3s cluster. Fedora 44 only — no Debian/Ubuntu, no `apt`, use `dnf`.

## Code style

- Simple, minimal solutions. No abstractions, variables, or conditionals for cases that don't exist yet.
- Prefer plain `kubectl`/`dnf`/`systemd` tasks over clever Ansible tricks.
- One role = one concern. Split roles/playbooks by step (see `ansible/roles/k3s-*`) rather than bundling unrelated tasks.
- Don't add fallbacks or error handling for failure modes that can't happen on this fixed, known inventory.

## Verifying changes

- Real hosts are reachable: `ansible-playbook -i ansible/inventory.ini <playbook> --syntax-check` first, then run for real — this is a live 5-node cluster, not a sandbox.
- Confirm with `kubectl get nodes` / `systemctl status` / log tails after changes, don't just assume a playbook run succeeded.
