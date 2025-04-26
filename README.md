# Ansible Playbooks for k3s Cluster Setup for Flux/Cilium

This repository contains Ansible playbooks to automate the setup and preparation of a k3s cluster, specifically designed for use with FluxCD and Cilium as the CNI.

The process involves two distinct phases managed by separate playbooks:

1.  **Setup (`setup_k3s_for_flux.yaml`):** Wipes nodes, installs k3s *with its default Flannel CNI*, installs the Flux CLI, and prepares the cluster for manual Flux bootstrapping.
2.  **Cleanup (`cleanup_flannel.yaml`):** Removes the initial Flannel CNI components *after* you have successfully bootstrapped Flux and verified that Cilium is installed and stable.

**Goal:** To achieve a minimal k3s cluster managed by Flux where Cilium provides networking, LoadBalancing (via BGP), and Ingress, starting from a state that allows Flux controllers to initialize correctly.

## Prerequisites

1.  **Ansible:** Ansible must be installed on the machine where you will run the playbooks.
2.  **SSH Access:** You need SSH access (preferably key-based) from your control machine to all target nodes (master and workers).
3.  **Sudo Privileges:** The SSH user needs `sudo` privileges on the target nodes (ideally configured for passwordless `sudo` for Ansible's `become`).
4.  **Python:** Target nodes need Python installed for Ansible modules to function (usually Python 2.7 or Python 3.x).
5.  **jq:** The `jq` package is used by the wipe script. The setup playbook attempts to install it, but pre-installing is recommended.
6.  **Inventory File:** An Ansible inventory file defining your master and worker nodes (e.g., `inventory.yaml`).
7.  **Flux GitOps Repository:** A separate Git repository prepared with your Flux configurations (like the `flux-k3s` repo structure discussed previously), including Cilium HelmRelease definitions in the `infrastructure` layer.
8.  **GitHub Personal Access Token (PAT):** Needed for the manual `flux bootstrap` step (with `repo` scope).

## Setup

1.  **Clone This Repository:**
    ```bash
    git clone git@github.com:sudoflux/ansible-flux-cni.git
    cd ansible-flux-cni
    ```

2.  **Configure Inventory (`inventory.yaml`):**
    *   Create or modify your Ansible inventory file.
    *   Ensure it defines a `[master]` group with **exactly one** host and a `[workers]` group.
    *   Set the `ansible_user` and potentially `ansible_ssh_private_key_file`.
    *   **Verify** the hostname used for the master in the inventory (e.g., `k3s-master1`) matches the `k3s_master_hostname` variable defined inside both playbooks.

3.  **Review Playbook Variables:**
    *   Check `k3s_master_hostname` in both YAML files.
    *   Check `target_user` and `target_user_home` in `setup_k3s_for_flux.yaml` if you need kubeconfig set up for a user other than `ansible_user`.
    *   Review the `k3s_flux_wipe_script` content within `setup_k3s_for_flux.yaml` to ensure it meets your needs.

## Workflow

**Phase 1: Cluster Setup**

1.  **Run the Setup Playbook:**
    ```bash
    ansible-playbook -i inventory.yaml setup_k3s_for_flux.yaml
    ```
    This playbook will:
    *   Wipe all nodes.
    *   Reboot all nodes.
    *   Install k3s server (with Flannel) on the master.
    *   Install k3s agent on workers.
    *   Install Flux CLI on the master.
    *   Copy kubeconfig to the specified user's home directory on the master.

2.  **Verify Initial Setup:**
    *   SSH into the master node (`k3s-master1` or your designated name).
    *   Run `kubectl get nodes`. Wait for all nodes to show `STATUS` as `Ready`. This confirms Flannel is working.
    *   You should see Flannel pods running: `kubectl get pods -n kube-system -l app=flannel`.

**Phase 2: Manual Flux Bootstrap & Verification (CRITICAL)**

1.  **Prepare for Bootstrap:**
    *   On the master node, export your GitHub Personal Access Token:
        ```bash
        export GITHUB_TOKEN="ghp_YourPersonalAccessTokenHere"
        ```

2.  **Run Flux Bootstrap:**
    *   Execute the `flux bootstrap` command, pointing it to your Flux GitOps repository and the correct path within it (e.g., `./clusters/k3s-home`).
        ```bash
        flux bootstrap github \
            --owner="sudoflux" \
            --repository="flux-k3s" \
            --branch=main \
            --path=./clusters/k3s-home \
            --personal
        ```

3.  **WAIT and VERIFY:** This is the most important step. Flux needs time to install Cilium, and Cilium needs time to initialize. Monitor actively:
    *   **Flux Sync Status:** `watch flux get kustomizations -A` and `watch flux get helmreleases -A`. Wait for `infrastructure` Kustomization and `cilium` HelmRelease to show `Ready = True`.
    *   **Cilium Pods:** `watch kubectl get pods -n kube-system -l k8s-app=cilium`. Ensure all pods become `Running`.
    *   **Node Status:** `watch kubectl get nodes`. Ensure all nodes remain or become `Ready` (this confirms Cilium CNI is active).
    *   **Cilium Status:** `cilium status --namespace kube-system`. Check for errors, verify BGP peering (if applicable).
    *   **Storage:** `kubectl get pv`. Verify your static PVs are `Available`.
    *   **Ingress Service:** `kubectl get svc -n kube-system <cilium-ingress-service-name>`. Verify `TYPE=LoadBalancer` and `EXTERNAL-IP` is assigned correctly.
    *   **(Optional) Test:** Deploy a simple test application and Ingress via Flux (by pushing to your apps layer) to confirm basic Cilium functionality.

    **Do NOT proceed until you are confident Cilium is fully functional and stable.** This may take 5-15 minutes or more after bootstrapping finishes.

**Phase 3: Flannel Cleanup**

1.  **Run the Cleanup Playbook:** Only after completing and verifying Phase 2:
    ```bash
    ansible-playbook -i inventory.yaml cleanup_flannel.yaml
    ```
    This playbook will:
    *   Find the Flannel DaemonSet using `kubectl`.
    *   Delete the Flannel DaemonSet.
    *   (Optional) Remove the Flannel CNI configuration file from `/etc/cni/net.d/` on all nodes.

## Outcome

After completing all phases successfully, you will have:

*   A k3s cluster running.
*   Cilium installed and managing CNI, LoadBalancing (BGP), and Ingress.
*   The initial Flannel CNI components removed.
*   Flux managing your cluster state based on your GitOps repository.
*   Your applications (defined in the apps layer of your Flux repo) deploying and using Cilium for networking/ingress and your NFS storage for persistence.
