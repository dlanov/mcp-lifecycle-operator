# Agentic Networking Example

This example demonstrates deploying an MCP server with the MCP Lifecycle
Operator and exposing it through
[kube-agentic-networking](https://github.com/kubernetes-sigs/kube-agentic-networking),
a Kubernetes SIG project that provides Gateway API-based routing for agent/MCP
traffic.

## What this demonstrates

- **MCP Lifecycle Operator** owns the lifecycle of the MCP server: it reconciles
  an `MCPServer` resource into a Deployment and a Service (`everything-mcp-server`,
  exposing port 3001). The MCP application behind that Service serves MCP
  traffic at the `/mcp` path.
- **kube-agentic-networking** routes to that Service through an `XBackend`. The
  `XBackend` references the Service by name/port/path, an `HTTPRoute` routes
  traffic from a `Gateway` to that `XBackend`, and the kube-agentic-networking
  controller programs an Envoy proxy accordingly.

```
MCPServer --(operator creates)--> Service (everything-mcp-server:3001/mcp)
                                        ^
                                        | referenced by
                                    XBackend (everything-mcp-backend)
                                        ^
                                        | backendRefs
                                    HTTPRoute (everything-mcp-route)
                                        ^
                                        | parentRefs
                                    Gateway (everything-mcp-gateway)
```

This example intentionally stops at routing. kube-agentic-networking also
supports MCP method-level authorization via `XAccessPolicy` — see
[Optional: tool-level authorization](#optional-tool-level-authorization) below
for a pointer, but no `XAccessPolicy` is included here.

## Prerequisites

- A Kubernetes cluster.
- **MCP Lifecycle Operator** CRDs installed and the controller running (`make install`,
  `make run` — see the [main README](../../README.md)).
- **kube-agentic-networking** installed according to its current official
  quickstart / installation instructions:
  <https://github.com/kubernetes-sigs/kube-agentic-networking>
  (hosted quickstart: <https://kube-agentic-networking.sigs.k8s.io/guides/quickstart/>).
  Installing it provisions the `kube-agentic-networking` `GatewayClass` used by
  this example, along with the CRDs and controller it depends on. Follow the
  upstream instructions in full rather than installing pieces individually —
  the setup does more than install CRDs and a Deployment.

## Deployment

1. Deploy the MCP server:

   ```bash
   kubectl apply -f examples/agentic-networking/mcpserver.yaml
   ```

2. Wait for it to become ready:

   ```bash
   kubectl wait --for=condition=Ready mcpserver/everything-mcp-server -n default --timeout=2m
   kubectl get mcpserver everything-mcp-server -n default
   ```

3. Deploy the Gateway API / kube-agentic-networking resources:

   ```bash
   kubectl apply -f examples/agentic-networking/networking.yaml
   ```

## Verification

### MCP Lifecycle Operator side

```bash
# MCPServer should show Ready=True and Accepted=True
kubectl get mcpserver everything-mcp-server -n default

# The operator-managed Service (created automatically for the MCPServer)
kubectl get svc everything-mcp-server -n default
```

### kube-agentic-networking side

```bash
kubectl get xbackend everything-mcp-backend -n default -o yaml
```

Confirm `spec.mcp` matches the Service above (`serviceName: everything-mcp-server`,
`port: 3001`, `path: /mcp`). XBackend currently does not expose a readiness
condition. Verify that its spec references the expected Service, and use the
Gateway and HTTPRoute status conditions below to verify routing reconciliation.

```bash
kubectl get gateway everything-mcp-gateway -n default -o yaml
# or, just the conditions and assigned address:
kubectl get gateway everything-mcp-gateway -n default \
  -o jsonpath='{.status.conditions}{"\n"}{.status.addresses}{"\n"}'
```

Look for `Programmed=True` and `Accepted=True` in `status.conditions`, and at
least one entry under `status.addresses`.

```bash
kubectl get httproute everything-mcp-route -n default -o yaml
```

Look at `status.parents[].conditions` for `Accepted=True` and
`ResolvedRefs=True`.

## Traffic verification through the Gateway

This example does not include a copy/paste `curl` command for sending an actual
MCP request through the Gateway. The current kube-agentic-networking quickstart
puts the Gateway listener behind kube-agentic-networking's built-in SPIFFE-based
mTLS identity system, which requires client certificates issued through the
Kubernetes `PodCertificateRequest`/`ClusterTrustBundle` APIs. Those APIs are
alpha and, in kube-agentic-networking's own development setup, require a
purpose-built kind cluster with `PodCertificateRequest`, `ClusterTrustBundle`,
and `ClusterTrustBundleProjection` feature gates enabled — not something a
generic cluster has by default.

To exercise real traffic through the Gateway (including issuing an identity for
your test client), follow kube-agentic-networking's own quickstart:
<https://kube-agentic-networking.sigs.k8s.io/guides/quickstart/>. Its "Bring
Your Own Agent" section documents how to point an existing workload at a
Gateway once that identity infrastructure is set up.

### Troubleshooting the MCP backend directly (bypasses the Gateway)

This checks that the MCP server itself is healthy — it does **not** exercise
kube-agentic-networking's routing or authorization path:

```bash
kubectl port-forward svc/everything-mcp-server -n default 3001:3001
```

Then connect with an MCP client at `http://localhost:3001/mcp`.

## Optional: tool-level authorization

kube-agentic-networking also supports fine-grained, MCP method-level
authorization through the `XAccessPolicy` resource — for example, allowing a
given `ServiceAccount` to call only specific tools (`tools/call` with specific
tool-name params) on the backend defined here. This example does not include
one. See:

- The `XAccessPolicy` API:
  <https://github.com/kubernetes-sigs/kube-agentic-networking/blob/main/api/v1alpha1/accesspolicy_types.go>
- A worked example targeting an `XBackend`:
  <https://github.com/kubernetes-sigs/kube-agentic-networking/blob/main/site-src/guides/quickstart/policy/e2e.yaml>

## Cleanup

```bash
kubectl delete -f examples/agentic-networking/networking.yaml
kubectl delete -f examples/agentic-networking/mcpserver.yaml
```
