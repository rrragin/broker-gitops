# Solace Event Broker Operator via Argo CD (demo setup)

Vanilla Kubernetes, Argo CD v3.x, app-of-apps. The admin password Secret is created manually with
`kubectl` (demo only; see "Moving beyond a demo" at the end).

```
operator/            pinned operator manifest (CRD, RBAC, Deployment) - operator image 1.4.2
brokers/base         PubSubPlusEventBroker CR (references the solace-broker-admin Secret)
brokers/overlays/    dev (standalone, developer mode), prod (HA, 30Gi, PDB, prune/delete protected)
apps/                root app, operator (wave 0), broker ApplicationSet (wave 1)
secrets/             reference copy of the Secret shape (not applied by Argo CD)
```

## Bootstrap

1. Replace `https://github.com/YOUR-ORG/solace-gitops.git` in `apps/*.yaml` and set the destination
   cluster `server` URLs in `apps/solace-brokers.yaml`.
2. Create the namespaces and admin password Secret (next section).
3. Push to Git, then once: `kubectl apply -n argocd -f apps/root.yaml`.

## Admin password Secret

The broker CR references a Secret named `solace-broker-admin`, so it must exist in each broker
namespace before the first sync. Create the namespaces yourself (Argo CD's `CreateNamespace=true` is
harmless if they already exist), then the Secrets.

```bash
# dev
kubectl create namespace solace-dev
kubectl create secret generic solace-broker-admin \
  --namespace solace-dev \
  --from-literal=username_admin_password='CHOOSE-A-DEV-PASSWORD'

# prod (switch kubectl context first if prod is a different cluster)
kubectl create namespace solace-prod
kubectl create secret generic solace-broker-admin \
  --namespace solace-prod \
  --from-literal=username_admin_password='CHOOSE-A-PROD-PASSWORD'
```

Notes:
- The key name `username_admin_password` is what the broker is expected to read; confirm it in the
  Operator Parameters Reference for your operator version.
- Argo CD does not manage these Secrets, so it neither prunes nor overwrites them.
- Changed the order and synced first? The broker will wait on the missing Secret. Create it, and the
  operator picks it up on its next reconcile.
- To change a password later: delete and re-create the Secret, then restart the broker pods.

## Verify

```bash
kubectl get pods -n pubsubplus-operator-system
kubectl get secret solace-broker-admin -n solace-dev
kubectl get pubsubpluseventbroker -A
kubectl get pods -n solace-prod        # HA: 3 pods (primary, backup, monitor)
```

## Operations

- Upgrade operator: replace `operator/deploy.yaml` with the new upstream release, review, merge.
- Upgrade broker: set `spec.image.tag` in the overlay; the operator performs the rolling upgrade.
- Resources: each HA pod needs at least 2 CPUs / 4 GB memory.
- Prod CR carries `Prune=false,Delete=false`; removing a broker is a deliberate manual step.

## Moving beyond a demo

Manually created Secrets are not reproducible from Git: a rebuilt cluster loses them. For real
environments, deliver the Secret through GitOps with Sealed Secrets, SOPS or External Secrets, and
keep `adminCredentialsSecret` pointing at the same name.
