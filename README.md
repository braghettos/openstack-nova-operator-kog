# openstack-nova-operator-kog

Krateo Operator Generator (KOG) packaging that turns OpenStack Nova
compute instances into native Kubernetes custom resources, validated
against [OVH Public Cloud](https://www.ovhcloud.com/en/public-cloud/)
as the managed OpenStack target.

`kubectl apply` an `Instance` CR &rarr; KOG's
[`oasgen-provider`](https://github.com/krateoplatformops/oasgen-provider)
and
[`rest-dynamic-controller`](https://github.com/krateoplatformops/rest-dynamic-controller)
reconcile it into a real VM on OVH.

## What's in here

```
chart/
  Chart.yaml
  values.yaml
  assets/
    server.yaml            # Hand-crafted Nova OAS subset (Servers)
  templates/
    configmap-server.yaml  # Bundles the OAS into a ConfigMap
    rd-server.yaml         # RestDefinition pointing at the ConfigMap
    auth-bridge-*.yaml     # Stateless nginx that rewrites
                           # Authorization: Bearer -> X-Auth-Token
  samples/
    nova-auth.yaml         # Secret + InstanceConfiguration CR
    test-server.yaml       # Sample Instance CR
scripts/
  get-token.sh             # Keystone v3 token fetcher; --secret mode
                           # emits a kubectl-apply-ready Secret
docs/
  e2e.md                   # Full OVH walkthrough on a kind cluster
  architecture.md          # Diagram + design rationale
```

## Quickstart

```bash
helm repo add krateo https://charts.krateo.io
helm install oasgen-provider krateo/oasgen-provider -n krateo-system --create-namespace

# Discover Nova URL + token from OVH Keystone
export OS_AUTH_URL=https://auth.cloud.ovh.net/v3
export OS_USERNAME=...
export OS_PASSWORD=...
export OS_PROJECT_ID=...
export OS_REGION_NAME=GRA11
./scripts/get-token.sh --upstream
# TOKEN=...
# NOVA_URL=https://compute.gra11.cloud.ovh.net/v2.1/<projectId>

helm install nova ./chart -n krateo-system \
  --set authBridge.upstreamUrl="$NOVA_URL"

./scripts/get-token.sh --secret | kubectl apply -f -
kubectl apply -f chart/samples/nova-auth.yaml
# edit chart/samples/test-server.yaml: flavorRef / imageRef / network UUID
kubectl apply -f chart/samples/test-server.yaml
kubectl -n krateo-system get instances.nova.openstack.krateo.io -w
```

See [docs/e2e.md](docs/e2e.md) for the full walkthrough including a
`kind` bootstrap and cleanup, and [docs/architecture.md](docs/architecture.md)
for the design rationale (specifically why a header-rewrite proxy is
needed today).

## Authentication note

KOG only accepts OpenAPI `http/basic` and `http/bearer` security
schemes. Nova authenticates with `X-Auth-Token`. This chart papers over
the gap with a stateless ~30-line nginx config that rewrites the header
before forwarding to OVH. **You** are responsible for keeping the
Keystone token in the Secret fresh (OVH default lifetime ~1h);
`scripts/get-token.sh --secret | kubectl apply -f -` is the supported
refresh path. When upstream KOG grows `apiKey` support, the bridge can
be removed.

## License

Apache-2.0 - see [LICENSE](LICENSE).
