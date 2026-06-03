# Instana client-IP-through-Apache test (LoadBalancer → Apache → helloworld)

Goal: **see the real client IP in an Instana trace** when traffic flows through a
load balancer and an Apache reverse proxy before reaching the application.

The whole point: *what do you have to configure in the Apache proxy?*

## Architecture (deployed to cluster master `192.168.178.35`, namespace `xff-test`)

```
 client (your Mac / LAN — arrives as 192.168.178.130 via the VPN gateway)
        │
        ▼   Service type=LoadBalancer  (MetalLB → 192.168.178.216)
        │   externalTrafficPolicy: Local      ← preserves the real source IP
  ┌─────────────┐
  │   nginx     │  L7 load balancer. Sets  X-Forwarded-For: <client>
  └─────────────┘  (this is what F5 / HAProxy / cloud ALB do)
        │
        ▼
  ┌─────────────┐  ★ THE KEY TIER ★
  │   Apache    │  mod_remoteip : RemoteIPHeader X-Forwarded-For
  │   httpd     │                 RemoteIPInternalProxy 10.244.0.0/16
  │             │  mod_proxy    : ProxyPass  (auto re-adds X-Forwarded-For)
  └─────────────┘
        │   forwards  X-Forwarded-For: 192.168.178.130
        ▼
  ┌─────────────┐  Node.js, auto-instrumented by Instana AutoTrace.
  │ helloworld  │  INSTANA_EXTRA_HTTP_HEADERS=x-forwarded-for;...
  └─────────────┘  → client IP captured as a tag on the HTTP span
```

Both tiers are auto-instrumented by the cluster's **instana-autotrace-webhook**
(no code/image changes). Traces land in `instanak3s.lab.allwaysbeginner.com`
under cluster `kubernetes`, namespace `xff-test`.

---

## ★ The answer: the Apache settings you need

`20-apache.yaml` mounts `proxy-xff.conf`. The essential lines:

```apache
LoadModule remoteip_module   modules/mod_remoteip.so
LoadModule proxy_module      modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

# 1) Tell Apache to read the client IP out of X-Forwarded-For ...
RemoteIPHeader        X-Forwarded-For
# 2) ... and which upstream hops to trust (here: the in-cluster pod network,
#        i.e. the nginx LB pod). Only these are stripped from the header.
RemoteIPInternalProxy 10.244.0.0/16

# 3) Reverse-proxy to the app. mod_proxy AUTOMATICALLY re-adds the (now
#    corrected) client IP to X-Forwarded-For on the way to helloworld.
ProxyPreserveHost On
ProxyPass        /  http://helloworld.xff-test.svc.cluster.local:3000/
ProxyPassReverse /  http://helloworld.xff-test.svc.cluster.local:3000/
```

After this, Apache's own request client IP (`%a`, `REMOTE_ADDR`, and what the
Instana Apache sensor reports) becomes the **real client**, and the app
downstream receives `X-Forwarded-For: <real-client>`.

### ⚠️ The two gotchas this lab actually uncovered (the important part)

1. **`RemoteIPInternalProxy` vs `RemoteIPTrustedProxy` for *private* clients.**
   `mod_remoteip` walks `X-Forwarded-For` right-to-left and, with
   **`RemoteIPTrustedProxy`**, it will **refuse to promote a private/RFC1918
   address** (10.x / 172.16.x / 192.168.x) to "client" — a *trusted external*
   proxy is not expected to report a private client. Proven here:

   | X-Forwarded-For sent to Apache | client Apache resolves |
   |---|---|
   | `8.8.8.8` (public)             | `8.8.8.8` ✅ |
   | `192.168.178.130` (private)    | *(refused — stays the proxy)* ❌ with TrustedProxy |
   | `192.168.178.130` (private)    | `192.168.178.130` ✅ with **InternalProxy** |

   Our real client arrives as a **private** LAN/VPN IP (`192.168.178.130`), so
   we must use **`RemoteIPInternalProxy`**. (If your clients are public Internet
   IPs, `RemoteIPTrustedProxy` is fine and slightly stricter.)

2. **Never trust the subnet your clients live in.** If you add the client's own
   range (e.g. `192.168.0.0/16`) to the proxy list, `mod_remoteip` treats the
   client as just another proxy and strips it — you lose the client IP. Trust
   **only** the actual upstream proxy hops (here the pod CIDR `10.244.0.0/16`).

### Two ways the client IP shows up in Instana

| Mechanism | What you configure | Where it shows |
|---|---|---|
| **Captured HTTP header** (robust, works for any client incl. private) | `INSTANA_EXTRA_HTTP_HEADERS=x-forwarded-for` env on the Node app (or agent-level `com.instana.tracing.extra-http-headers`) | A tag on the HTTP entry call: `X-Forwarded-For = 192.168.178.130` |
| **Connection client IP** on the Apache span | `mod_remoteip` (above) so Apache's `REMOTE_ADDR` = real client | The Apache/httpd call's client |

---

## Files

| File | Purpose |
|---|---|
| `00-namespace.yaml` | namespace `xff-test` (auto-traced) |
| `10-helloworld.yaml` | Node.js echo app + `INSTANA_EXTRA_HTTP_HEADERS` |
| `20-apache.yaml`    | **Apache reverse proxy — the key config** |
| `30-lb-nginx.yaml`  | nginx L7 LB + MetalLB LoadBalancer (`192.168.178.216`, `externalTrafficPolicy: Local`) |
| `40-apache-direct.yaml` | **Test B:** Apache as the *direct edge* proxy, no L7 LB (`192.168.178.217`); proves mod_proxy alone sets XFF + strips spoofed XFF |

## Deploy

```bash
kubectl apply -f 00-namespace.yaml -f 10-helloworld.yaml -f 20-apache.yaml -f 30-lb-nginx.yaml
kubectl -n xff-test get pods,svc
```

## Test

```bash
# Full chain — response shows what helloworld received:
curl -s http://192.168.178.216/hello | grep -E 'clientIp|xForwardedFor'
#   "xForwardedFor": "192.168.178.130"
#   "clientIp": "192.168.178.130"

# Confirm Apache resolved the real client (mod_remoteip):
kubectl -n xff-test logs deploy/apache -c apache | tail
#   client=192.168.178.130  xff="-"  "GET /hello HTTP/1.0"  200
```

## Test B — Apache as edge, NO load balancer (`40-apache-direct.yaml`)

A second path where the client hits Apache **directly** (no nginx L7 tier). The
L4 `Service` (`192.168.178.217`) only forwards TCP; Apache is the edge proxy and
sets `X-Forwarded-For` itself. No `mod_remoteip` needed (no inbound XFF to read).
The edge setting `RequestHeader unset X-Forwarded-For early` discards any
client-supplied XFF so it can't be spoofed.

```bash
# Normal request -> helloworld receives the real client IP
curl -s http://192.168.178.217/ | grep -E 'clientIp|xForwardedFor'
#   "xForwardedFor": "192.168.178.130"   "clientIp": "192.168.178.130"

# Spoof attempt -> the fake header is stripped, real client still forwarded
curl -s -H "X-Forwarded-For: 1.2.3.4" http://192.168.178.217/ | grep -E 'clientIp|xForwardedFor'
#   "xForwardedFor": "192.168.178.130"   (1.2.3.4 discarded)
```

Verified 2026-06-02: both return `192.168.178.130`; apache-direct log shows
`client=192.168.178.130  xff-in="-"`. Key difference vs Test A: at the edge,
`mod_proxy` (`ProxyAddHeaders On`) is what sets XFF — `mod_remoteip` is only
needed when there is an upstream proxy whose XFF must be interpreted.

## See it in Instana

UI: `https://instanak3s.lab.allwaysbeginner.com` → Applications / Services →
**helloworld** (namespace `xff-test`). Open a call → the captured
**`X-Forwarded-For`** request header shows `192.168.178.130` = the real client.
Generate load first: `for i in $(seq 1 60); do curl -s http://192.168.178.216/shop/$i >/dev/null; done`

## Verified results (2026-06-02)

- Apache access log resolves real client: `client=192.168.178.130` ✅
- helloworld receives clean `X-Forwarded-For: 192.168.178.130` ✅
- Instana Node collector `agentready`, capturing `x-forwarded-for` ✅
- `RemoteIPInternalProxy` required (not `TrustedProxy`) for the private client ✅

## Cleanup

```bash
kubectl delete namespace xff-test
```
# xforwardefor-test
