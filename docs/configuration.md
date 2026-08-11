---
type: Configuration
title: openstack-nova-operator-kog — configuration
description: The whole values surface — the per-resource RestDefinition toggles, the nova-auth-bridge proxy (upstream URL, image, service, resources, scheduling), and the serviceAccount and name overrides — all typed by values.schema.json.
resource: oci://ghcr.io/krateo-blueprints/charts/openstack-nova-operator-kog
tags: [configuration, values, restdefinitions, auth-bridge]
timestamp: 2026-08-11T00:00:00Z
---

# Configuration

Everything is `chart/values.yaml`, fully typed by `chart/values.schema.json`. The
schema requires `authBridge`, and `authBridge.upstreamUrl` in particular — the chart
hard-fails on an empty upstream URL. When the chart is installed as a Krateo
Composition, the same keys are the spec of the generated `OpenstackNovaOperatorKog` CR
([api](./api.md)).

## RestDefinition toggles (`restdefinitions.*`)

One block per Nova resource. Each emits (when enabled) a ConfigMap holding the OAS
subset plus a `RestDefinition` that oasgen-provider turns into a CRD and controller.

| resource | key | `enabled` default | generated `resourceKind` |
|---|---|---|---|
| Nova Servers (instances) | `restdefinitions.server` | `true` | `Instance` |
| Nova Flavors | `restdefinitions.flavor` | `true` | `ComputeFlavor` |
| Nova Keypairs | `restdefinitions.keypair` | `true` | `ComputeKeypair` |
| Nova Server Groups | `restdefinitions.server_group` | `true` | `ComputeServerGroup` |
| Nova Host Aggregates | `restdefinitions.aggregate` | `true` | `ComputeAggregate` |
| Nova Quota Sets | `restdefinitions.quota` | `false` | `ComputeQuota` |

Every block carries the same three keys:

| key | default | effect |
|---|---|---|
| `enabled` | per table above | emit the ConfigMap + RestDefinition for this resource. |
| `resourceGroup` | `nova.openstack.krateo.io` | the Kubernetes API group of the generated CRD; the GVK becomes `<resourceGroup>/v1alpha1`. |
| `resourceKind` | per table above | the generated CRD Kind. **Prefixed on purpose** — a Kind matching the envelope property name (`Server`, `Flavor`, ...) collides in `crdgen` and yields an invalid schema, so the kinds are `Instance`, `ComputeFlavor`, etc. |

`quota` is off by default: Nova's `PUT /os-quota-sets/{tenant_id}` rejects the
`tenant_id` path field leaking into the body (`additionalProperties: false`), which
needs a KOG path-only / excluded-from-body mapping that has not landed yet. See
`chart/assets/quota.yaml`.

## The auth bridge (`authBridge.*`)

`nova-auth-bridge` is a tiny stateless nginx reverse proxy that rewrites
`Authorization: Bearer <token>` (emitted by rest-dynamic-controller) into
`X-Auth-Token: <token>` (what Nova expects). It does **no** Keystone token exchange —
you fetch and store the token out-of-band ([usage](./usage.md)).

| key | default | effect |
|---|---|---|
| `authBridge.enabled` | `true` | deploy the proxy (Deployment + Service + ConfigMap). |
| `authBridge.replicaCount` | `1` | number of proxy replicas (`minimum: 0`). |
| `authBridge.upstreamUrl` | `""` | **THE key input** — the region-specific Nova endpoint from your Keystone catalog, e.g. `https://compute.gra11.cloud.ovh.net/v2.1/<projectId>`. Required; empty fails the install. |
| `authBridge.image.repository` | `nginx` | proxy image repository. |
| `authBridge.image.tag` | `1.27-alpine` | proxy image tag. |
| `authBridge.image.pullPolicy` | `IfNotPresent` | one of `Always`, `IfNotPresent`, `Never`. |
| `authBridge.service.type` | `ClusterIP` | one of `ClusterIP`, `NodePort`, `LoadBalancer`. |
| `authBridge.service.port` | `80` | in-cluster Service port (1–65535). |
| `authBridge.resources.requests` | `cpu 10m` / `memory 16Mi` | container requests. |
| `authBridge.resources.limits` | `cpu 200m` / `memory 64Mi` | container limits. |
| `authBridge.podAnnotations` | `{}` | extra pod annotations. |
| `authBridge.nodeSelector` | `{}` | node selector for the proxy pod. |
| `authBridge.tolerations` | `[]` | tolerations for the proxy pod. |
| `authBridge.affinity` | `{}` | affinity rules for the proxy pod. |

Note the upstream Nova host must be a resolvable, reachable URL from inside the
cluster. For an in-cluster (Krateo-blueprint) Nova, point the bridge at the `nova-api`
**ClusterIP** rather than a DNS name — a DNS name trips the nginx resolver
([e2e-krateo-openstack](./e2e-krateo-openstack.md)).

## ServiceAccount and overrides

| key | default | effect |
|---|---|---|
| `serviceAccount.create` | `false` | create a ServiceAccount for chart workloads. |
| `serviceAccount.name` | `""` | name to use; empty + `create: false` uses the namespace `default` SA. |
| `nameOverride` | `""` | override the chart name used in resource names. |
| `fullnameOverride` | `""` | override the fully-qualified app name used in resource names. |
