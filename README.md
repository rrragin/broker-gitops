# Solace Event Broker Operator via Argo CD

Vanilla Kubernetes, Argo CD v3.x, app-of-apps.

```
operator/            pinned operator manifest (CRD, RBAC, Deployment) - operator image 1.4.2
brokers/base         PubSubPlusEventBroker CR
brokers/overlays/    dev (standalone, developer mode), prod (HA, 30Gi, PDB, prune/delete protected)
apps/                root app, operator app (wave 0), broker ApplicationSet (wave 1)
secrets/             example admin Secret (do not commit real values)
```

## Bootstrap

1. Replace `https://github.com/YOUR-ORG/solace-gitops.git` in `apps/*.yaml` and set the destination
   cluster `server` URLs in `apps/solace-brokers.yaml`.
2. Push to Git, then once: `kubectl apply -n argocd -f apps/root.yaml`.
3. Deliver the admin Secret (see `secrets/`) into each broker namespace and uncomment
   `adminCredentialsSecret` in `brokers/base/eventbroker.yaml`.

## Verify

```bash
kubectl get pods -n pubsubplus-operator-system
kubectl get pubsubpluseventbroker -A
kubectl get pods -n solace-prod        # HA: 3 pods (primary, backup, monitor)
```

## Operations

- Upgrade operator: replace `operator/deploy.yaml` with the new upstream release, review, merge.
- Upgrade broker: set `spec.image.tag` in the overlay; the operator performs the rolling upgrade.
- Resources: each HA pod needs at least 2 CPUs / 4 GB memory.
- Prod CR carries `Prune=false,Delete=false`; removing a broker is a deliberate manual step.
