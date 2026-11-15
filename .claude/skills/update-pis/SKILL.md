---
name: update-pis
description: Update the OS packages on the Raspberry Pi cluster nodes and reboot them using os-update.yaml, then verify the k3s cluster came back healthy. Use when the user asks to update, upgrade or patch the Pis/raspberries/nodes.
---

# Update the Raspberry Pis

`os-update.yaml` runs `apt update`, upgrades all packages, does a `dist-upgrade`, cleans the apt cache and reboots. It targets the `cluster` group (`pi1` master, `pi2`–`pi5` workers) and reboots **all nodes at the same time**, so the cluster and its services (PiHole DNS on `192.168.4.202`, ingress, qbittorrent) are down until every node is back.

## Steps

1. **Check reachability first**:
   ```shell
   ansible cluster -m ansible.builtin.ping
   ```
   If a node is unreachable, stop and report it; don't update the rest without asking.

2. **Pick the scope.** If the user didn't say, ask whether to update all nodes at once (with a short full outage) or one node at a time. For one at a time, update the workers first and `pi1` last, and wait for each node to be `Ready` before moving on:
   ```shell
   ansible-playbook os-update.yaml --limit pi2
   ```

3. **Run the update**:
   ```shell
   ansible-playbook os-update.yaml            # all nodes
   ansible-playbook os-update.yaml --limit <node>
   ```
   Updates plus a reboot can take several minutes per node, so use a long timeout or run it in the background.

4. **Verify the cluster**:
   ```shell
   ssh pi1 kubectl get nodes -o wide
   ssh pi1 kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   ```
   All nodes should be `Ready`. Pods may take a minute or two to settle after the reboot, especially Longhorn volumes; check again before you call anything broken.

5. **Report** which nodes were updated, the play recap (`changed`/`failed` per host), the kernel/node versions from `kubectl get nodes -o wide`, and any pods that aren't healthy.

## Notes

- The updates come from apt only; k3s itself isn't upgraded here. Its version is pinned as `k3s_version` in `config.yaml` and installed by `main.yaml`.
- If a play fails partway, report the failing host and task from the output before retrying.
