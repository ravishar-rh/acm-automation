# acm-automation

Deploy the Red Hat Advanced Cluster Management (ACM) operator on a given OpenShift cluster via **ArgoCD** (pull method) or **Ansible**.

## Contents

- **manifests/acm-operator/** – Kubernetes manifests (Namespace, OperatorGroup, Subscription, MultiClusterHub) for the ACM operator, managed with Kustomize.
- **argocd/** – ArgoCD Application that syncs the ACM operator manifests to a target cluster.
- **ansible/** – Ansible playbook that applies the same manifests to a cluster using `kubectl apply -k`.

## Prerequisites

- **Target cluster**: OpenShift with OLM and the `redhat-operators` catalog (default on OpenShift).
- **ArgoCD**: For the ArgoCD path, ArgoCD must be installed (this creates the `argocd` namespace) and have access to the target cluster. If the `argocd` namespace is missing, install ArgoCD first, then apply the Application.
- **Ansible**: For the Ansible path, `kubectl` (or `oc`) and `kustomize` must be in `PATH`; `KUBECONFIG` or `kubeconfig` extra var points at the target cluster.

---

## 1. Deploy with ArgoCD

1. Ensure the **`argocd` namespace exists** so the Application apply does not fail. If you see `namespaces "argocd" not found`:
   ```bash
   oc apply -f argocd/namespace.yaml
   ```
   Then install Argo CD (or OpenShift GitOps) into that namespace so the Application is actually synced. **OpenShift**: install the [OpenShift GitOps](https://docs.openshift.com/container-platform/latest/cicd/gitops/installing-openshift-gitops.html) operator (it usually uses the `openshift-gitops` namespace — if so, use that namespace for the Application: set `metadata.namespace: openshift-gitops` in **argocd/application-acm-operator.yaml** and apply the Application).

2. Update the Application source in **argocd/application-acm-operator.yaml**:
   - Set `source.repoURL` to your Git repo (e.g. this repo).
   - Set `source.path` to `manifests/acm-operator` (or your path).
   - Set `source.targetRevision` if not `HEAD`.

3. Set the **destination cluster**:
   - **In-cluster**: Keep `destination.server: https://kubernetes.default.svc`.
   - **Remote cluster**: Add the cluster in ArgoCD (`argocd cluster add <context>`) and set either:
     - `destination.name: <cluster-name>`, or  
     - `destination.server: <cluster-api-url>`.

4. Apply the Application:

   ```bash
   kubectl apply -f argocd/application-acm-operator.yaml
   ```

5. ArgoCD will sync and create the `open-cluster-management` namespace, OperatorGroup, Subscription, and **MultiClusterHub** CR. The ACM operator installs via OLM; the MultiClusterHub CR triggers deployment of the full ACM hub (console, cluster management, MCE).

---

## 2. Deploy with Ansible

1. Point at your cluster:
   - Set `KUBECONFIG` to the kubeconfig of the target cluster, or  
   - Pass it when running the playbook: `-e "kubeconfig=/path/to/kubeconfig"`.

2. Run the playbook from the **ansible** directory (or pass the path to the playbook and inventory):

   ```bash
   cd ansible
   ansible-playbook -i inventory/sample.yml playbook-deploy-acm-operator.yml
   ```

   With a specific kubeconfig:

   ```bash
   ansible-playbook -i inventory/sample.yml playbook-deploy-acm-operator.yml -e "kubeconfig=/path/to/kubeconfig"
   ```

3. The playbook runs `kubectl apply -k` against **manifests/acm-operator**, creating the namespace, OperatorGroup, Subscription, and MultiClusterHub. The ACM operator installs via OLM; the MultiClusterHub triggers the full hub install.

---

## 3. Cleanup (Ansible)

To remove the ACM operator and related resources from a cluster:

1. Use the same inventory and kubeconfig as for deploy.

2. Run the cleanup playbook:

   ```bash
   cd ansible
   ansible-playbook -i inventory/sample.yml playbook-cleanup-acm-operator.yml
   ```

   With a specific kubeconfig:

   ```bash
   ansible-playbook -i inventory/sample.yml playbook-cleanup-acm-operator.yml -e "kubeconfig=/path/to/kubeconfig"
   ```

The cleanup playbook:

1. Deletes any **MultiClusterHub** and **MultiClusterEngine** CRs (so the operator can tear down managed components).
2. Waits briefly for the operator to start teardown.
3. Deletes the ACM operator manifests (Subscription, OperatorGroup, and the `open-cluster-management` namespace) with `kubectl delete -k`.

If the namespace sticks (e.g. finalizers), remove any remaining resources in that namespace and, if needed, patch the namespace to remove finalizers before re-running or delete the namespace manually.

---

## Customization

- **ACM channel**: The Subscription `spec.channel` must match a channel in your cluster’s `redhat-operators` catalog. If you see **"no operators found in channel release-2.11"** (or similar), the channel does not exist in your catalog. List available channels and the default:
  ```bash
  oc get packagemanifest advanced-cluster-management -n openshift-marketplace -o jsonpath='{.status.defaultChannel}'
  oc get packagemanifest advanced-cluster-management -n openshift-marketplace -o jsonpath='{range .status.channels[*]}{.name}{"\n"}{end}'
  ```
  Then set `spec.channel` in **manifests/acm-operator/subscription.yaml** to one of those (e.g. `release-2.14`, `release-2.15`, or the defaultChannel). The manifest default is `release-2.14`.
- **Install plan**: Set `spec.installPlanApproval` in the Subscription to `Automatic` or `Manual`.
