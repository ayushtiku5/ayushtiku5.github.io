---
layout: post
title: "A Tiny Gateway for Enforcing Service-to-Service Policy"
date: 2026-07-25 10:00:00 -0000
categories: systems
---

Most "API gateway" tutorials are really about north-south traffic: a client hits the gateway, the gateway authenticates it and forwards it to one backend. I wanted to prototype something closer to what a service mesh's sidecar proxy does: east-west traffic, where every call between internal services passes through a policy check before it's allowed to happen. So I built a small Go gateway that sits in front of three toy services and enforces an explicit allow/deny table between them. Code is here: [github.com/ayushtiku5/api-gateway](https://github.com/ayushtiku5/api-gateway).

## The setup: three services that don't fully trust each other

There's `user-service`, `order-service`, and `inventory-service`, each a minimal HTTP server. `order-service` needs to call `user-service` to enrich an order with the buyer's info, and it needs to call `inventory-service` to reserve stock when an order is created. Nothing calls `inventory-service` or `user-service` back, and `inventory-service` and `user-service` never talk to each other at all. That asymmetry is exactly what the policy table is meant to enforce, rather than something services are just trusted to respect on their own.

```yaml
policies:
  - source: order-service
    target: user-service
    action: allow
  - source: order-service
    target: inventory-service
    action: allow
  - source: user-service
    target: order-service
    action: allow
  - source: user-service
    target: inventory-service
    action: deny
  - source: inventory-service
    target: user-service
    action: deny
  - source: inventory-service
    target: order-service
    action: deny

default_action: deny
```

No service calls another directly. Every inter-service call goes through the gateway first, identifying itself with two headers:

```go
func callViaGateway(method, targetService, path string, body io.Reader) (*http.Response, error) {
	req, err := http.NewRequest(method, gatewayURL+path, body)
	...
	req.Header.Set("X-Service-Name", serviceName)
	req.Header.Set("X-Target-Service", targetService)
	return http.DefaultClient.Do(req)
}
```

## The policy engine: a map lookup, not a rules DSL

I was tempted to build something more elaborate here, wildcard matching, regex on paths, scoped permissions per route. I didn't. The engine is a `map[string]string` keyed by `"source->target"`, checked once per request:

```go
func (e *PolicyEngine) Check(source, target string) bool {
	key := fmt.Sprintf("%s->%s", source, target)
	action, ok := e.rules[key]
	if !ok {
		action = e.defaultAction
	}
	return action == "allow"
}
```

Anything not explicitly listed falls through to `default_action`, which is `deny` unless the config says otherwise. That default matters more than the rules themselves: it means adding a fourth service to this system doesn't silently open up every path between it and everything else. It starts with zero permissions and has to be granted access explicitly, the same posture you'd want from a real mesh's authorization policy.

## The proxy: check first, forward second

The HTTP handler is small enough to read as a single flow: pull the two headers, check the policy, look up the target's address, and only then hand off to a reverse proxy.

```go
func (h *ProxyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	source := r.Header.Get("X-Service-Name")
	target := r.Header.Get("X-Target-Service")

	if source == "" || target == "" {
		writeJSON(w, http.StatusBadRequest, ...)
		return
	}

	if !h.engine.Check(source, target) {
		writeJSON(w, http.StatusForbidden, map[string]string{
			"error": "policy denied", "source": source, "target": target,
		})
		log.Printf("[GATEWAY] DENY  %s -> %s %s %s", source, target, r.Method, r.URL.Path)
		return
	}

	targetBase, ok := h.services[target]
	...
	proxy := httputil.NewSingleHostReverseProxy(targetURL)
	...
	r.Header.Del("X-Service-Name")
	r.Header.Del("X-Target-Service")
	proxy.ServeHTTP(w, r)
}
```

Two details worth calling out. First, a denied request never reaches `httputil.NewSingleHostReverseProxy`, the policy check is a hard gate before any proxying logic runs, not a filter layered on top of it. Second, the routing headers get stripped before the request is forwarded, so `inventory-service` never sees `X-Service-Name: order-service` on the request it receives; it just sees a plain HTTP call. The identity claim is meaningful only at the gateway boundary, which is also the only place it's checked.

Every decision gets logged with the direction, method, path, and (for allowed calls) the upstream's response code and latency, so `ALLOW order-service -> inventory-service PUT /inventory/prod-1/reserve 200 (4ms)` and `DENY user-service -> inventory-service` both show up clearly in one place instead of being scattered across each service's own logs.

## Seeing it deny something

The `make demo-denied` target is the part that actually demonstrates the policy working, not just being configured:

```
=== [1] user-service -> inventory-service (DENY by policy) ===
HTTP status: 403

=== [3] unknown-service -> user-service (DENY, no matching rule, default=deny) ===
HTTP status: 403
```

That third case is the one I actually cared about proving: a service name that isn't in the policy table at all still gets a 403, not a 502 or an accidental allow. The default-deny posture holds even for a caller the config has never heard of.

## What this isn't

There's no mTLS or any cryptographic verification that a request claiming `X-Service-Name: order-service` actually came from `order-service`, the header is just trusted at face value, which is fine for a local prototype and not fine for anything crossing a real network boundary. There's also no per-route granularity within a policy, `order-service -> user-service` is all-or-nothing rather than scoped to specific paths or methods. Both are the obvious next layer if this stopped being a demo, real service identity (SPIFFE/mTLS or a signed token) instead of a plain header, and path-level policy instead of service-level. But the core question I wanted answered, does a centralized default-deny policy gate actually change what "denied" looks like at the wire level versus just being a config that nothing enforces, is answered by that 403.
