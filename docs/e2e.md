# End-to-end walkthrough on OVH Public Cloud

This guide walks through deploying the chart against a real OVH Public
Cloud project, from a clean machine with no Kubernetes cluster.

## Prerequisites

- An OVH Public Cloud project (you'll need its `OS_PROJECT_ID`, the
  32-character hex tenant id).
- An OpenStack user inside that project (created from the OVH manager
  or via `openstack user create`). **Not** your `ovhcloud.com` login.
- Local tools: `docker` (or any container runtime), `kind`, `kubectl`,
  `helm`, `curl`, `jq`.

## 1. Bootstrap a kind cluster

```bash
kind create cluster --name nova-kog
kubectl create namespace krateo-system
```

## 2. Install the Krateo OASGen Provider

```bash
helm repo add krateo https://charts.krateo.io
helm repo update
helm install oasgen-provider krateo/oasgen-provider \
  --namespace krateo-system
```

Wait for the controller to be ready:

```bash
kubectl -n krateo-system rollout status deploy/oasgen-provider
```

## 3. Discover your Nova endpoint and fetch a Keystone token

Source your OpenStack credentials. The simplest is to export them:

```bash
export OS_AUTH_URL=https://auth.cloud.ovh.net/v3
export OS_USERNAME=<openstack-user>
export OS_PASSWORD=<openstack-password>
export OS_PROJECT_ID=<32hex-project-id>
export OS_REGION_NAME=GRA11    # or SBG5, DE1, BHS5, WAW1, ...
```

Run the helper to get the token **and** the region-specific Nova URL:

```bash
./scripts/get-token.sh --upstream
# TOKEN=gAAAAA...
# NOVA_URL=https://compute.gra11.cloud.ovh.net/v2.1/<projectId>
```

Capture both - you'll need `NOVA_URL` to install the chart and `TOKEN`
to seed the Secret.

## 4. Install this chart

```bash
helm install nova ./chart \
  --namespace krateo-system \
  --set authBridge.upstreamUrl="https://compute.gra11.cloud.ovh.net/v2.1/<projectId>"
```

Verify:

```bash
kubectl -n krateo-system get restdefinitions
kubectl -n krateo-system get deploy
# Expect: nova-openstack-nova-operator-kog-auth-bridge   1/1
# Wait for the RestDefinition to reach Ready=True - that means the
# generated Server CRD has been installed and the rest-dynamic-controller
# for it has been deployed.
kubectl get crd | grep nova.openstack.krateo.io
```

## 5. Provision a server

Push the Keystone token into a Secret (refresh whenever it expires;
OVH's default lifetime is ~1h):

```bash
./scripts/get-token.sh --secret | kubectl apply -f -
```

Apply the BearerAuth CR:

```bash
kubectl apply -f chart/samples/nova-auth.yaml
```

Pick a flavor / image / network from your project:

```bash
openstack flavor list
openstack image  list
openstack network list
```

Edit `chart/samples/test-server.yaml` with those values, then:

```bash
kubectl apply -f chart/samples/test-server.yaml
kubectl -n krateo-system get servers.nova.openstack.krateo.io -w
```

Cross-check on OVH:

```bash
openstack server show kog-demo-1
```

## 6. Cleanup

```bash
kubectl delete -f chart/samples/test-server.yaml
helm -n krateo-system uninstall nova
helm -n krateo-system uninstall oasgen-provider
kind delete cluster --name nova-kog
```

## Token refresh

The chart's `nova-auth-bridge` is intentionally stateless and does no
Keystone exchange. When the Secret's token expires:

```bash
./scripts/get-token.sh --secret | kubectl apply -f -
```

The next reconciliation will pick up the new token automatically.

## Troubleshooting

- **`401 Unauthorized` from Nova**: the token in the Secret is expired
  or wasn't scoped to the right project. Re-run `get-token.sh --secret`.
- **`404` on `POST /servers`**: confirm `authBridge.upstreamUrl` includes
  the `/v2.1/<projectId>` suffix.
- **Server stays `pending` in `kubectl get`**: the RestDefinition hasn't
  reconciled yet. `kubectl describe restdefinition` and check the
  `oasgen-provider` logs.
