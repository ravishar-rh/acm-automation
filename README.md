# acm-automation

Deploy the Red Hat Advanced Cluster Management (ACM) operator on a given OpenShift cluster via **ArgoCD** (pull method) or **Ansible**.

## Contents

- **manifests/acm-operator/** – Kubernetes manifests (Namespace, OperatorGroup, Subscription) for the ACM operator, managed with Kustomize.
- **argocd/** – ArgoCD Application that syncs the ACM operator manifests to a target cluster.
- **ansible/** – Ansible playbook that applies the same manifests to a cluster using `kubectl apply -k`.

## Prerequisites

- **Target cluster**: OpenShift with OLM and the `redhat-operators` catalog (default on OpenShift).
- **ArgoCD**: For the ArgoCD path, ArgoCD must be installed (this creates the `argocd` namespace) and have access to the target cluster. If the `argocd` namespace is missing, install ArgoCD first, then apply the Application.
- **Ansible**: For the Ansible path, `kubectl` (or `oc`) and `kustomize` must be in `PATH`; `KUBECONFIG` or `kubeconfig` extra var points at the target cluster.

---

## 1. Deploy with ArgoCD

1. Ensure **ArgoCD is installed** on the cluster (the `argocd` namespace must exist). If you see `namespaces "argocd" not found`, install ArgoCD first:
   - **OpenShift**: Install the [OpenShift GitOps](https://docs.openshift.com/container-platform/latest/cicd/gitops/installing-openshift-gitops.html) operator (Red Hat Argo CD); it creates the `argocd` namespace.
   - **Other clusters**: Create the `argocd` namespace and install [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/#installation), e.g. from the official install manifest.

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

5. ArgoCD will sync and create the `open-cluster-management` namespace, OperatorGroup, and Subscription. The ACM operator will install via OLM. Optionally create a **MultiClusterHub** CR to deploy the full ACM hub.

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

3. The playbook runs `kubectl apply -k` against **manifests/acm-operator**, creating the same namespace, OperatorGroup, and Subscription. The ACM operator installs via OLM.

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

- **ACM channel**: Edit **manifests/acm-operator/subscription.yaml** and set `spec.channel` (e.g. `release-2.11`, `release-2.10`) or override via Kustomize vars if you add them.
- **Install plan**: Set `spec.installPlanApproval` in the Subscription to `Automatic` or `Manual`.
