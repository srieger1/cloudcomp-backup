# Lab 7: Argo CD and GitOps

## Overview

In this lab, you will learn how to deploy and manage Kubernetes applications with Argo CD, a GitOps controller for Kubernetes. You will:

- Install and explore Argo CD
- Deploy the Taskflow Helm chart from our Harbor OCI registry
- Use Git as the source of truth for application configuration
- Observe how Argo CD detects and corrects drift in the cluster

**Important:** For the GitOps part of this lab, use a Git service that is convenient for your group. Our self-hosted GitLab at [https://git-ce.rwth-aachen.de](https://git-ce.rwth-aachen.de) is the recommended default because it keeps the lab self-contained, but GitHub also works if you prefer it.

---

## Task 1: Understanding Argo CD and GitOps

### 1.1: What Is GitOps?

GitOps is a way of managing infrastructure and applications where Git is the single source of truth.

In a GitOps workflow:

- You declare the desired state in Git
- A controller watches Git and the cluster
- The controller continuously reconciles the live cluster to match what is in Git

This is different from manually changing resources with `kubectl edit`, `kubectl patch`, or the Kubernetes dashboard. Those changes may work temporarily, but they are not durable unless they are also committed to Git.

Argo CD implements the GitOps model for Kubernetes. It watches your Git repository, compares it with the live state in the cluster, and reports whether they match. Alternatives to ArgoCD include Flux CD.

### 1.2: Install Argo CD

If Argo CD is not already installed in your cluster, install it into its own namespace:

```bash
helm install --create-namespace --set server.service.type=LoadBalancer -n argocd argocd oci://ghcr.io/argoproj/argo-helm/argo-cd
```

Wait until the Argo CD pods are running:

```bash
kubectl get pods -n argocd
```

### 1.3: Access the Argo CD UI

Argo CD exposes a web UI. For this lab, use the UI.

Get the floating IP (`EXTERNAL-IP`) of Argo CD, 

```bash
kubectl get svc argocd-server -n argocd
```

or if not running in OpenStack, use port-forward to be able to access Argo CD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Open https://floating-ip (or [http://localhost:8080](http://localhost:8080) when using port-forward) in your browser and log in with: (accept the self-signed certificate for now)

- Username: `admin`
- Password: the value you just retrieved

Explore the UI and familiarize yourself with the different views.

## Task 2: Deploying the Taskflow Helm Chart with Argo CD

In this task, you will deploy the Taskflow application from our Harbor OCI registry using Argo CD.
This is the same chart reference used in Lab 5, but now Argo CD will manage it continuously instead of a one-time `helm install`.

### 2.1: Discover the Chart Version

Argo CD needs a chart tag as the `targetRevision`. Use Helm to inspect the chart metadata and find the published version:

```bash
helm show chart oci://harbor.cs.hs-fulda.de/cloud-computing/helm/taskflow
```

Look for the `version` field (not `appVersion`) in the output. Use that version in the Application manifest below.

### 2.2: Create a Git Repository for the Lab

Create a new repository (`lab7-argocd-taskflow`) in GitLab or GitHub for this lab. A simple structure is enough:

```text
lab7-argocd-taskflow/
  applications/
    taskflow.yaml
```

If you use GitLab, create a blank project in [https://git-ce.rwth-aachen.de](https://git-ce.rwth-aachen.de).
Make sure that the project/repository is **Public**. Clone it locally, and add the files from this lab.

### 2.3: Create the Argo CD Application

Create a file named `applications/taskflow.yaml` in your repository (commit and push):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: taskflow
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: harbor.cs.hs-fulda.de/cloud-computing/helm
    chart: taskflow
    targetRevision: 0.1.0 # <-- replace with the actual chart version you found
  destination:
    name: in-cluster
    namespace: taskflow
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Understanding the configuration:

- `metadata.finalizers` ensures that when you delete the Application, all managed resources are cleaned up properly
- `repoURL` points to the OCI Helm registry where the Taskflow chart is published
- `chart` is the name of the chart to deploy
- `targetRevision` is the chart tag or version you found with `helm show chart`
- `destination.name: in-cluster` tells Argo CD to deploy to the same cluster where it is running
- `destination.namespace` tells Argo CD where to deploy the application
- `automated.prune: true` removes resources that disappear from Git
- `automated.selfHeal: true` brings the cluster back if someone changes resources manually
- `CreateNamespace=true` lets Argo CD create the namespace automatically

### 2.4: Bootstrap the Application

Apply the Application manifest to the cluster:

```bash
kubectl apply -f applications/taskflow.yaml
```

#### Hands-On Exercises

1. **Watch the application appear:** Open the Argo CD UI and look for `taskflow`.

2. **Check sync status:** The application should move from `OutOfSync` to `Synced` once Argo CD applies the chart.

3. **Inspect the resources:** Use the Argo CD UI to see the resources created by the application, Deployments, Services, etc.

---

## Task 3: GitOps in Practice

The key idea in GitOps is that the cluster should match Git, not the other way around. In this task, you will make a manual change in the live cluster and observe how Argo CD responds. Then we will connect a real Git repository and make a change there to see the effect.

### 3.1: Create Drift on Purpose

First, inspect the workload names in the Taskflow namespace:

```bash
kubectl get deploy,sts,svc -n taskflow
```

Choose one managed resource, for example the frontend Deployment, and delete it from the live cluster:

```bash
kubectl delete deployment taskflow-frontend -n taskflow
```

If you prefer, you can delete a Service or another managed resource instead.

### 3.2: Observe Reconciliation

Watch the Application in the Argo CD UI. Because the Application is configured with automated sync and self-heal, Argo CD should recreate the missing resource.

#### Questions

1. What status does Argo CD show immediately after you delete the resource?

2. How long does it take before the resource comes back?

3. Why is this behavior useful in a production system?

### 3.3: Connect to Git

Until now, we have been applying the Application manifest directly to the cluster using `kubectl apply`. This is fine for testing, but in a real GitOps workflow, you would commit the manifest to Git and let Argo CD watch it.
When Argo CD is configured to watch a Git repository, it will automatically detect changes in Git and apply them to the cluster. This means that if you edit the Application manifest in Git and push the change, you never touch the cluster directly. Argo CD will update the live application accordingly.

For this part, make sure your Application manifest is in a Git repository that Argo CD can access. If you followed the earlier steps, you should have already created this repository and pushed the manifest there.

First, let's delete the existing Application that points to the OCI registry, since we will replace it with a new one that points to Git:

```bash
kubectl delete application taskflow -n argocd
```

You can also do this through the Argo CD UI by selecting the application and choosing "Delete". Make sure to choose "Foreground" propagation so that all managed resources are cleaned up.

We will need to create a new Application `bootstrap.yaml` that points to the Git repository instead of the OCI registry. It will be the entrypoint for managing all the applications from Git and is usually the first and only Application you create outside of Git.
Be sure to update the `repoURL` and `path` fields to match your repository structure.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://git-ce.rwth-aachen.de/<user-slug>/lab7-argocd-taskflow.git
    targetRevision: HEAD
    path: applications
  destination:
    name: in-cluster
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Understanding the new configuration:

- `repoURL` now points to your Git repository instead of the OCI registry
- `targetRevision: HEAD` tells Argo CD to track the latest commit on the default branch
- `path` points to the directory in your repository where the Application manifests are located (in this case, `applications`)

Apply this new Application manifest to the cluster:

```bash
kubectl apply -f bootstrap.yaml
```

Observe that Argo CD now shows the `bootstrap` application, and it should sync successfully, creating the `taskflow` application as a child.

### 3.4: Change the Desired State in Git

Now make a real Git change by editing the Application manifest in your repository. Good examples include:

- Changing `destination.namespace` to deploy the same chart into a different namespace
- Change the helm chart values by adding them under `source.helm.valuesObject` in the Application manifest ([see Argo CD documentation for details](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#values))

Commit and push the change to your Git repository.

Observe the change in the Argo CD UI. Wait for the next reconciliation cycle of Argo CD. The application should update to match the new Git state.

---

## Feeling adventurous?

If you want to extend this lab, try one of the following:

1. Add a second Argo CD Application that deploys the same chart into a `staging` namespace.
2. Put the Application manifests into a GitLab project and use branch-based environments such as `main` and `dev`.

These patterns are common in real GitOps setups and build naturally on the concepts introduced in this lab.

---

## Cleanup

When you are done, remove the `bootstrap` Application. This will also remove the `taskflow` application and all its resources because of the finalizer we added.

```bash
kubectl delete application -n argocd bootstrap
```

We can also uninstall Argo CD if you want to free up resources:

```bash
helm uninstall -n argocd argocd
```

---

## Summary

- Argo CD is a GitOps controller for Kubernetes
- Git is the source of truth for desired state
- Argo CD keeps the live cluster aligned with Git
- Manual changes in the cluster are temporary if they are not reflected in Git

This workflow is useful because it makes deployments auditable, repeatable, and easier to roll back.
