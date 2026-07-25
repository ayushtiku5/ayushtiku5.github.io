---
layout: post
title: "Building a Reverse Proxy from Scratch in Go"
date: 2026-07-25 12:00:00 -0000
categories: systems
---

I've spent a lot of time operating other people's proxies: Envoy configs, GCLB, the odd nginx box that everyone's afraid to touch. At some point I wanted to understand the mechanics well enough to build the thing myself rather than just configuring it, so I put together a small reverse proxy prototype in Go. Code is here: [github.com/ayushtiku5/reverse-proxy](https://github.com/ayushtiku5/reverse-proxy).

The goal wasn't to compete with Envoy or Traefik. It's a learning project, built in phases, each one adding a layer of what a "real" proxy needs. This post walks through what's there so far.

## The core: routing + load balancing

At the center is a single `RouteProxy` per route, wrapping Go's `httputil.ReverseProxy`. Each incoming request goes through a `Router` that matches on the longest matching path prefix, so `/api/v2` wins over `/api` if both are registered, and hands off to the balancer for that route's backend pool.

```go
func (rp *RouteProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	backend, err := rp.balancer.Next(r)
	...
	proxy := &httputil.ReverseProxy{
		Transport: rp.transport,
		Director: func(req *http.Request) {
			req.URL.Scheme = target.Scheme
			req.URL.Host = target.Host
			...
		},
	}
	proxy.ServeHTTP(w, r)
}
```

A new `Director` closure is created per request so the backend address chosen by the balancer is captured correctly under concurrent load, but the underlying `http.RoundTripper` is shared across requests, so connection pooling to upstreams still works as intended.

Load balancing is pluggable: the config picks a strategy by name and everything implements a common `Balancer` interface:

- **round_robin**: cycles through healthy backends
- **weighted**: smooth weighted round-robin (the same algorithm nginx uses: each backend accumulates its weight every round, the highest current weight wins and gets debited by the total weight), which spreads traffic evenly instead of bursting toward the heaviest backend
- **least_conn**: routes to whichever backend has the fewest active connections, tracked with an atomic counter
- **random**: uniform random pick among healthy backends

All four are safe for concurrent use and support atomic backend-list swaps, which matters for the next piece.

## Phase 2: active health checking

A background `Checker` probes every backend on an interval (`GET /health`, configurable path/interval/timeout), and a `Registry` tracks consecutive failures per address. Once an address crosses the configured failure threshold, the registry flips it unhealthy and pushes an updated backend list into the balancer atomically, so in-flight requests aren't affected and the balancer never has to know a health check even happened.

```yaml
health_check:
  enabled: true
  path: "/health"
  interval: "2s"
  timeout: "1s"
  threshold: 2
```

This was the fun part to get right: the balancer, the checker, and the registry all need to agree on the *current* backend list without a shared lock across the whole request path. The registry owns that synchronization and the balancers just consume whatever list they're handed.

## Phase 3: TLS, mTLS, and rate limiting

The proxy can terminate TLS itself, and optionally require client certificates (mTLS) by pointing `client_ca` at a CA bundle; `tls.RequireAndVerifyClientCert` does the enforcement. There's a `certs/gen-certs.sh` script in the repo to spin up a throwaway CA plus server/client cert pairs for local testing.

Rate limiting sits in front of the proxy as middleware, built on `golang.org/x/time/rate` token buckets, with two independent dimensions that can both be active at once:

- **per-IP**: one bucket per client address, created lazily and cached
- **per-route**: one shared bucket across all clients hitting that route

Route-level config can override the global per-IP limit and add a route-wide cap on top:

```yaml
rate_limit:
  per_ip:
    rate: 5
    burst: 10
  per_route:
    rate: 50
    burst: 100
```

## What's still a placeholder

I built this in explicit phases and I'm being upfront that not everything is wired up yet. Metrics, tracing, circuit breaking, retries, and per-route header rewriting all have their files scaffolded in `internal/middleware/` with the shape of the eventual API, but the bodies are stubs: literally just a comment noting which phase they're slated for. The middleware chain is built with a `Chain()` helper that composes handlers left-to-right, so wiring in a new middleware later is additive; it slots into `buildHandler` without touching the phases already working.

Also already in place: structured JSON access logging (method, path, status, bytes, latency, request ID) on every request, and zero-downtime config reload. The server swaps its whole handler graph via an `atomic.Value` so a config change doesn't drop connections.

## Testing it

There's a tiny `cmd/backend`, a bare HTTP server with `/`, `/health`, `/slow`, and `/headers` endpoints, that stands in for real upstreams. The Makefile spins up three of them and points the proxy at all three, then has targets to poke at specific behavior:

```
make run-all        # 3 backends + proxy on phase-1 config
make test-lb         # confirm requests cycle across backends
make test-ratelimit  # confirm 429s show up after the burst
make test-mtls       # confirm client-cert auth works
```

Nothing fancy, but enough to convince myself the load balancer actually balances and the rate limiter actually limits, rather than just reading like it should.

## Why bother

None of this is novel. Every one of these pieces exists, better tested and more battle-hardened, in Envoy or nginx or Traefik. But building the toy version is what made the tradeoffs click: why smooth weighted round-robin exists instead of naive weighted random, why health state needs to flow through a registry instead of living inside each balancer, why an atomic handler swap is the cheap way to get hot reload without a restart. That's the kind of thing that's hard to internalize from reading someone else's YAML schema; you have to hit the sharp edges yourself.

More phases (metrics, circuit breaking, retries) to come.
