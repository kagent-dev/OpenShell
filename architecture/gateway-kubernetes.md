# Gateway Kubernetes Install

This document describes the minimum operator-managed Kubernetes deployment for OpenShell when you already have a cluster and want to install the gateway with Helm.

## Scope

This path targets a plaintext MVP install:

- Helm deploys the gateway into your cluster.
- The gateway uses its in-cluster ServiceAccount to manage sandbox CRs.
- Sandbox pods side-load the `openshell-sandbox` supervisor from a dedicated supervisor image via an init container and `emptyDir` volume.
- CLI clients register the gateway with `openshell gateway add http://<host>:<port>`.

mTLS hardening for cluster-deployed gateways is a follow-up step.

## Install

Create a values file with the minimum settings for an externally reachable plaintext gateway:

```yaml
image:
  repository: ghcr.io/nvidia/openshell/gateway
  tag: latest

supervisor:
  image:
    repository: ghcr.io/nvidia/openshell/supervisor
    tag: latest

server:
  disableTls: true
  disableGatewayAuth: true
  grpcEndpoint: http://openshell.openshell.svc.cluster.local:8080
  sshGatewayHost: gateway.example.com
  sshGatewayPort: 8080
```

Install the chart:

```shell
helm install openshell deploy/helm/openshell -n openshell --create-namespace -f values.yaml
```

The chart creates the gateway `StatefulSet`, ServiceAccount, namespaced RBAC for sandbox CRs and event watches, and a cluster-scoped read-only permission on `nodes` for GPU capacity checks.

## Supervisor Delivery

Sandbox pods do not rely on a node `hostPath`.

The Kubernetes driver injects:

1. An `emptyDir` volume named `openshell-supervisor-bin`.
2. An init container named `openshell-supervisor-loader`.
3. A read-only mount of that volume at `/opt/openshell/bin` in the agent container.
4. A command override to `/opt/openshell/bin/openshell-sandbox`.

The init container copies `/openshell-sandbox` from `ghcr.io/nvidia/openshell/supervisor:<tag>` into the shared volume before the agent container starts. This keeps the supervisor version-locked with the gateway release without requiring sandbox images or node images to embed the binary.

## Reaching the Gateway

For development, port-forward the Service:

```shell
kubectl -n openshell port-forward svc/openshell 8080:8080
```

For shared use, expose the Service through a `NodePort`, `LoadBalancer`, or `Ingress`, then set `server.sshGatewayHost` and `server.sshGatewayPort` to the externally reachable address in your Helm values.

Register the gateway from the client:

```shell
openshell gateway add http://<reachable-host>:<port>
```

## Minimum RBAC

The current gateway code path requires only these Kubernetes permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openshell
  namespace: openshell
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: openshell-sandbox
  namespace: openshell
rules:
  - apiGroups: ["agents.x-k8s.io"]
    resources: ["sandboxes"]
    verbs: ["create", "delete", "get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: openshell-sandbox
  namespace: openshell
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: openshell-sandbox
subjects:
  - kind: ServiceAccount
    name: openshell
    namespace: openshell
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: openshell-cluster
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshell-cluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshell-cluster
subjects:
  - kind: ServiceAccount
    name: openshell
    namespace: openshell
```

The gateway does not currently require `pods`, `pods/exec`, `pods/log`, `pods/status`, `secrets`, or `runtimeclasses` permissions for this install path.

## Next Steps

- Enable TLS and gateway authentication before exposing the gateway beyond a trusted environment.
- Add your preferred Service exposure strategy, such as `LoadBalancer` or `Ingress`.
- Set `server.sandboxImage` to your default sandbox image if you do not want the community base image.
