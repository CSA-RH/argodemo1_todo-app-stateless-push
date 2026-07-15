# Deploy Stateless Workload with ACM using ApplicationSet Push Model

Deploy a stateless application (`todo-app`) to a managed cluster using
the ACM ApplicationSet push model, then failover the application to a
different cluster by changing the Placement label in the Git repository.

> **Disclaimer:** This demo is for learning and testing purposes only.
> Automatic failover behavior, tolerations, and prioritizers should be
> thoroughly tested and configured in a non-production environment
> before applying to any production cluster.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Hub Cluster (openshift-gitops namespace)                        │
│                                                                  │
│  ManagedClusterSetBinding ──► Placement ──► PlacementDecision    │
│                          (label: env=prod)                       │
│                                    │                             │
│                                    ▼                             │
│                            GitOpsCluster                         │
│                   (register clusters in ArgoCD as                │
│                    cluster secrets + creates the                 │
│                    acm-placement ConfigMap)                      │
│                                    │                             │
│                                    ▼                             │
│  ApplicationSet (clusterDecisionResource generator)              │
│        │  reads ConfigMap ──► knows how to parse                 │
│        │                      PlacementDecision                  │
│        │  reads PlacementDecision ──► gets cluster names         │
│        │  reads ArgoCD cluster secrets ──► gets server URLs      │
│        │                                                         │
│        └──► Application: todo-app-cluster1                       │
│              destination: {{server}} (cluster1 API URL)          │
└──────────────────────────────────────────────────────────────────┘
          │
          ▼
   ┌──────────────┐          ┌──────────────┐
   │   cluster1   │          │   cluster2   │
   │ (env: prod)  │          │ (env: dev)   │
   │              │          │              │
   │  ✔ selected  │          │  ✘ not       │
   │  todo-app    │          │    selected  │
   │  deployed    │          │              │
   └──────────────┘          └──────────────┘
```

---

## Procedure

### 1. Prerequesites

Make sure the Hub cluster has the OpenShift-Gitops operator installed and the ACM topology is created. If not first configure it as per the following document and configure the GitOpsCluster CR for push model: 

```bash
https://github.com/CSA-RH/acm_bootstrap/blob/main/README.md
```

Get the kubeconfig of both managed clusters

```bash
CLUSTER_NAME=cluster1
SECRET_NAME=$(oc get secret -n $CLUSTER_NAME -o name | grep 'admin-kubeconfig')
oc extract $SECRET_NAME -n $CLUSTER_NAME --keys=kubeconfig --to=- > /tmp/${CLUSTER_NAME}-kubeconfig

CLUSTER_NAME=cluster2
SECRET_NAME=$(oc get secret -n $CLUSTER_NAME -o name | grep 'admin-kubeconfig')
oc extract $SECRET_NAME -n $CLUSTER_NAME --keys=kubeconfig --to=- > /tmp/${CLUSTER_NAME}-kubeconfig
```

### 2. Clone repo

```bash
cd /tmp
git clone https://github.com/CSA-RH/argodemo1_todo-app-stateless-push.git
cd argodemo1_todo-app-stateless-push
```

### 3. Create the Placement that will select the managed clusters to deploy the application

```bash
oc create -f bootstrap/placement-todo-app.yaml
```


### 4. Create the ApplicationSet

```bash
oc create -f argocd/applicationset-push.yaml
```

### 5. Verify the deployment

**Check ArgoCD Applications and ApplicationSet**

```bash
oc get applicationsets.argoproj.io -n openshift-gitops 
oc get applications.argoproj.io -n openshift-gitops | grep todo-app
```

Expected output:
```
todo-app-cluster1   Synced   Healthy
```

**Check the workloads on the managed cluster**

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get deployment,route -n todo-app
```

**Check the ACM console**

Navigate to **Applications → todo-app**. The topology view should show green
status indicators for all resources.

---

## Failover

> **Stateless vs Stateful:** This demo covers **stateless** failover,
> where ACM can automatically redeploy the application from Git on a
> healthy cluster. For **stateful** applications (with persistent data),
> automatic failover is not supported, failover is always
> user-initiated to prevent data loss. Stateful DR requires additional
> tooling such as OpenShift DR (Ramen), VolSync, or OADP to replicate
> persistent data before switching over.

### Automatic failover (cluster fault)

When a managed cluster becomes unreachable, ACM automatically taints it
with `NoSelect`. If the `Placement` does not tolerate that taint (as in
this demo), the cluster is evicted from the `PlacementDecision` and the
`ApplicationSet` redeploys the application on another matching cluster
, no Git change needed.

A `Placement` decides which clusters a workload lands on in two steps:
first it **filters** (which clusters are eligible), then it
**prioritizes** (which of those are best). Prioritizers score each
eligible cluster and pick the winners. This demo does not configure
prioritizers, but more details can be found in the
[ACM 2.17 Clusters documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/clusters)
and the
[Placement PrioritizerPolicy reference](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/clusters/index#placement-prioritizerpolicy).

### Manual failover (prod to dev)

To failover, patch the `Placement` on the hub cluster to change the
label selector from `prod` to `dev`. The `Placement` re-evaluates,
updates the `PlacementDecision`, and the `ApplicationSet` automatically
removes the application from `prod` clusters and deploys it on `dev`
clusters.

This is a **stateless failover**, the application is redeployed from
Git on the new cluster. No persistent data is migrated.

```
 ┌─────────────────────────────────────────────────────────┐
 │ 1. oc patch placement (prod → dev)                      │
 │                      │                                  │
 │                      ▼                                  │
 │    Placement re-evaluates → new PlacementDecision       │
 │                      │                                  │
 │                      ▼                                  │
 │    ApplicationSet detects change                        │
 │      ├── deletes Application: todo-app-cluster1 (prod)  │
 │      └── creates Application: todo-app-cluster2 (dev)   │
 │                      │                                  │
 │           ┌──────────┴──────────┐                       │
 │           ▼                     ▼                       │
 │    cluster1 (prod)       cluster2 (dev)                 │
 │    app REMOVED           app DEPLOYED                   │
 └─────────────────────────────────────────────────────────┘
```

### 6.1. Verify current state

```bash
# Which clusters the Placement currently selects
oc get placementdecisions -n openshift-gitops \
  -l cluster.open-cluster-management.io/placement=todo-app-placement \
  -o jsonpath='{range .items[*]}{.status.decisions[*].clusterName}{"\n"}{end}'

# Which ArgoCD Applications exist
oc get applications.argoproj.io -n openshift-gitops | grep todo-app
```

### 6.2. Patch the Placement to select `dev` clusters

```bash
oc patch placement todo-app-placement -n openshift-gitops --type merge \
  -p '{"spec":{"predicates":[{"requiredClusterSelector":{"labelSelector":{"matchExpressions":[{"key":"environment","operator":"In","values":["dev"]}]}}}]}}'
```

The Placement re-evaluates. Since the selector now matches
`environment: dev`, ACM updates the `PlacementDecision`:

- Clusters with `environment: prod` are **removed** from the decision
- Clusters with `environment: dev` are **added** to the decision

The `ApplicationSet` detects the `PlacementDecision` change and:

- **Deletes** the ArgoCD Application for the old `prod` cluster(s).
  Because `prune: true` is enabled, ArgoCD removes all workload
  resources from those clusters.
- **Creates** a new ArgoCD Application for each `dev` cluster.
  ArgoCD syncs and deploys the todo-app on the new clusters.

### 6.3. Verify the failover

```bash
# Confirm the new PlacementDecision
oc get placementdecisions -n openshift-gitops \
  -l cluster.open-cluster-management.io/placement=todo-app-placement \
  -o jsonpath='{range .items[*]}{.status.decisions[*].clusterName}{"\n"}{end}'

# Confirm the new ArgoCD Application
oc get applications.argoproj.io -n openshift-gitops | grep todo-app

# Confirm workloads on the new cluster
CLUSTER_NAME=cluster2
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get deployment,route -n todo-app
```

---
