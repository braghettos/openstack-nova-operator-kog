# Quickstart — Nova (compute) operator

Manage OpenStack **Nova** compute as Kubernetes CRs. End to end: install the
operator, `kubectl apply` an `Instance`, and watch the VM appear in the Horizon
dashboard.

## 1. Prerequisites

Krateo's KOG provider in the cluster:

```bash
helm repo add krateo https://charts.krateo.io && helm repo update
helm upgrade --install oasgen-provider krateo/oasgen-provider -n krateo-system --create-namespace
```

A reachable Nova endpoint. The chart ships an `auth-bridge` that rewrites
`Authorization: Bearer <token>` → `X-Auth-Token`; you supply a Keystone token
(see `scripts/get-token.sh`). For an in-cluster (Krateo-blueprint) Nova, point the
bridge at the `nova-api` **ClusterIP** (a DNS name trips the nginx resolver):

```bash
NOVA_IP=$(kubectl -n openstack get svc nova-api -o jsonpath='{.spec.clusterIP}')
PROJECT_ID=$(openstack project show admin -f value -c id)
helm upgrade --install nova-kog ./chart -n krateo-system \
  --set authBridge.upstreamUrl="http://$NOVA_IP:8774/v2.1/$PROJECT_ID"
kubectl -n krateo-system wait restdefinition/nova-kog-server --for=condition=Ready --timeout=300s
```

## 2. Supply a token + create an Instance

```bash
./scripts/get-token.sh --secret | kubectl apply -f -      # nova-token Secret
kubectl apply -f chart/samples/nova-auth.yaml             # InstanceConfiguration

# edit flavorRef / imageRef / network uuid for your cloud, then:
kubectl apply -f chart/samples/test-server.yaml
kubectl -n krateo-system get instances.nova.openstack.krateo.io -w
```

Within seconds the `Instance` CR is `Ready=True` and the VM is `ACTIVE` — the CR
records the server it created in `status.server.id`.

## 3. See it in Horizon

The instances the operator booted appear under **Compute → Instances**, `ACTIVE`,
on a Neutron network:

![Operator-created instances in Horizon](images/krateo-openstack/horizon-instances.png)

Beyond `Instance`, the operator also manages `ComputeFlavor`, `ComputeKeypair`,
`ComputeServerGroup` and `ComputeAggregate` the same way.
