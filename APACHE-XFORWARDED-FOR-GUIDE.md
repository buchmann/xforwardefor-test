# Apache `X-Forwarded-For` configuration guide
### Getting the real client IP into Apache logs and Instana traces, behind a load balancer

**Scope of this document.** A LoadBalancer terminates the client TCP connection
and forwards the request to Apache, which reverse-proxies to an application.
By default every downstream hop sees only the IP of the hop in front of it, so
the application — and Instana — record a *proxy* IP, not the real client. This
guide explains exactly which Apache directives fix that, what each parameter
does, and the non-obvious pitfalls.

---

## 1. The problem in one picture

```
                          Each TCP hop only knows its immediate peer:
 client 192.168.178.130
        │  TCP src = 192.168.178.130
        ▼
 ┌──────────────┐
 │ Load Balancer│   peer it sees: the client
 │   (nginx)    │
 └──────────────┘
        │  TCP src = 10.244.203.158  (the LB pod)   ← Apache's peer is the LB, NOT the client
        ▼
 ┌──────────────┐
 │   Apache     │   Without config: REMOTE_ADDR = 10.244.203.158  ✗
 └──────────────┘
        │  TCP src = 10.244.203.134  (the Apache pod)
        ▼
 ┌──────────────┐
 │ application  │   Without config: client looks like the Apache pod  ✗
 └──────────────┘
```

The fix is the **`X-Forwarded-For` (XFF)** HTTP header: each proxy records the IP
it saw and appends it to the header, so the chain is preserved in-band. Apache
needs to be told to (a) **read** that header to recover the real client, and
(b) **re-forward** it to the application.

---

## 2. The configuration (`proxy-xff.conf`)

This file is `Include`-d from the stock `httpd.conf`. It is the complete,
deployed config — every line is explained in §3.

```apache
# --- 1. Load the modules (ship with httpd:2.4, disabled by default) ---
LoadModule proxy_module        modules/mod_proxy.so
LoadModule proxy_http_module   modules/mod_proxy_http.so
LoadModule remoteip_module     modules/mod_remoteip.so
LoadModule headers_module      modules/mod_headers.so

# --- 2. Recover the real client IP from X-Forwarded-For ---
RemoteIPHeader        X-Forwarded-For
RemoteIPInternalProxy 10.244.0.0/16

# --- 3. Prove it: log the corrected client IP next to the raw header ---
LogFormat "client=%a  xff=\"%{X-Forwarded-For}i\"  \"%r\"  %>s" xff
CustomLog /proc/self/fd/1 xff

# --- 4. Reverse-proxy to the app (mod_proxy re-adds the corrected XFF) ---
ProxyPreserveHost On
ProxyAddHeaders   On
ProxyPass        /  http://helloworld.xff-test.svc.cluster.local:3000/
ProxyPassReverse /  http://helloworld.xff-test.svc.cluster.local:3000/

# --- 5. Belt-and-braces: also send X-Real-IP = corrected client IP ---
RequestHeader set X-Real-IP "expr=%{REMOTE_ADDR}"
```

---

## 3. Parameter reference

### 3.1 `LoadModule` — enable the modules

| Module | Why it is needed |
|---|---|
| `mod_proxy` + `mod_proxy_http` | The reverse-proxy engine (`ProxyPass`). `mod_proxy_http` is the HTTP backend transport. It also **auto-appends** the client IP to `X-Forwarded-For` when forwarding. |
| `mod_remoteip` | Reads `X-Forwarded-For` and **rewrites Apache's notion of the client IP** so logs, `REMOTE_ADDR`, and the Instana httpd sensor all report the real client. |
| `mod_headers` | Lets you set/append request headers (used here for `X-Real-IP`). |

> In the `httpd:2.4` image these modules exist but their `LoadModule` lines are
> commented out, so you must enable them explicitly.

---

### 3.2 `mod_remoteip` — the core of the solution

#### `RemoteIPHeader X-Forwarded-For`
- **What it does:** names the request header that carries the client chain.
  When a request arrives, `mod_remoteip` parses this header and replaces the
  connection's client IP (`r->useragent_ip`, exposed as `%a` in logs,
  `REMOTE_ADDR` to apps/CGI, and the client IP the Instana httpd sensor reports)
  with the recovered real client.
- **Value:** `X-Forwarded-For` is the de-facto standard. Use `X-Real-IP` or a
  custom header if your LB sets a different one. Use the RFC 7239 `Forwarded`
  header with `RemoteIPHeader Forwarded` only if your LB emits that format.
- **Note:** it changes the *logged/displayed* client. It does **not** change the
  real TCP socket; the original peer is still available as `%{c}a` in logs.

#### `RemoteIPInternalProxy 10.244.0.0/16`  ⭐ the one that trips people up
- **What it does:** declares which **upstream hops are trusted** to have set
  `X-Forwarded-For`. `mod_remoteip` walks the XFF list **right-to-left**,
  discarding addresses that match this list (they are known proxies), and stops
  at the **first address that is *not* in the list** — that becomes the client.
- **What to put here:** *only the proxy hop(s) that connect to Apache*, never the
  range your real clients live in. In Kubernetes the peer that reaches Apache is
  the LB pod, so the **pod CIDR `10.244.0.0/16`** is exactly right. Add more
  ranges/IPs (or repeat the directive) if you have multiple proxy tiers.

**`RemoteIPInternalProxy` vs `RemoteIPTrustedProxy` — pick the right one:**

| Directive | Trusts the proxy? | Will it accept a **private/RFC1918** client (10/172.16/192.168)? | Use when |
|---|---|---|---|
| `RemoteIPInternalProxy` | yes | **Yes** | Clients may be on private networks (corporate LAN, VPN, internal apps) |
| `RemoteIPTrustedProxy`  | yes | **No** — a private client is refused and Apache keeps the proxy IP | Clients are public Internet IPs and you want the stricter check |

> **This was the decisive finding in this lab.** With `RemoteIPTrustedProxy`,
> a public `8.8.8.8` in XFF was promoted to client, but the real private client
> `192.168.178.130` was *rejected* — Apache kept logging the LB pod IP. Switching
> to `RemoteIPInternalProxy` made Apache correctly report `192.168.178.130`.
> Because clients here arrive as a **private** LAN/VPN address, `InternalProxy`
> is required.

**Two failure modes to avoid:**
1. **Trusting the client's own subnet.** If you list e.g. `192.168.0.0/16` and
   your client is `192.168.178.130`, `mod_remoteip` treats the client as just
   another proxy, strips it, and you lose the client IP. Trust **only** the
   proxy hop (the pod CIDR).
2. **Using `TrustedProxy` for private clients.** See the table above.

#### Related `mod_remoteip` directives (not used here, good to know)
| Directive | Purpose |
|---|---|
| `RemoteIPProxiesHeader X-Forwarded-By` | Record the stripped proxy IPs into a separate header instead of discarding them. |
| `RemoteIPTrustedProxyList <file>` | Load a long trusted-proxy list from a file. |

---

### 3.3 Logging — how to *prove* it works

```apache
LogFormat "client=%a  xff=\"%{X-Forwarded-For}i\"  \"%r\"  %>s" xff
CustomLog /proc/self/fd/1 xff
```

| Token | Meaning |
|---|---|
| `%a` | **Client IP as `mod_remoteip` resolved it** — the real client after rewrite. |
| `%{c}a` | (alternative) the *raw* TCP peer, i.e. the LB pod — handy to log both for comparison. |
| `%{X-Forwarded-For}i` | The raw inbound `X-Forwarded-For` header value (`i` = inbound request header). |
| `%r` | The request line (`GET /path HTTP/1.1`). |
| `%>s` | Final HTTP status. |
| `CustomLog /proc/self/fd/1 xff` | Writes this log to **stdout** so it appears in `kubectl logs`. |

Expected line for a working setup:
```
client=192.168.178.130  xff="-"  "GET /hello HTTP/1.0"  200
```
`xff="-"` because `mod_remoteip` consumed the header after recovering the client.
If you instead see `client=10.244.203.158  xff="192.168.178.130"`, the rewrite
did **not** happen (wrong directive or untrusted proxy).

---

### 3.4 `mod_proxy` — forward to the app and re-add XFF

| Directive | What it does |
|---|---|
| `ProxyPass / http://helloworld...:3000/` | Reverse-proxy all requests to the backend. **`mod_proxy_http` automatically appends the current client IP to `X-Forwarded-For`** on the outbound request — and since `mod_remoteip` already set the client to the real IP, the backend receives `X-Forwarded-For: <real-client>`. It also adds `X-Forwarded-Host` and `X-Forwarded-Server`. |
| `ProxyPassReverse / ...` | Rewrites `Location`/redirect headers coming back from the backend so they point at Apache, not the internal service name. |
| `ProxyPreserveHost On` | Forwards the **original `Host`** header to the backend instead of the backend's hostname. Keeps virtual-host/routing logic and logged hostnames correct. |
| `ProxyAddHeaders On` | Explicitly enables sending the `X-Forwarded-*` headers to the backend (this is the default; set here for clarity). Set `Off` to stop Apache adding them. |

---

### 3.5 `mod_headers` — optional `X-Real-IP`

```apache
RequestHeader set X-Real-IP "expr=%{REMOTE_ADDR}"
```
- **What it does:** sets a single-value `X-Real-IP` header to the corrected
  client IP (`REMOTE_ADDR`, which `mod_remoteip` already fixed) before the
  request goes to the backend.
- **Why:** `X-Forwarded-For` can be a comma-separated list; `X-Real-IP` gives the
  app one unambiguous value. Many apps/log formats prefer it.
- `expr=%{REMOTE_ADDR}` uses Apache's expression parser to read the current
  client IP at request time.

---

## 4. What the header looks like at each hop (working setup)

| Stage | `X-Forwarded-For` value | Apache `REMOTE_ADDR` / client |
|---|---|---|
| Client → nginx | *(none)* | — |
| nginx → Apache | `192.168.178.130` | peer is the nginx pod `10.244.203.158` |
| `mod_remoteip` runs | header consumed | **`192.168.178.130`** ✅ |
| Apache → app (`mod_proxy`) | `192.168.178.130` (re-added) | — |
| App receives | `X-Forwarded-For: 192.168.178.130` | `clientIp = 192.168.178.130` ✅ |

---

## 5. How this surfaces in Instana

There are two independent ways the client IP reaches an Instana trace:

| Mechanism | Where you configure it | What you get |
|---|---|---|
| **Captured HTTP header** (works for *any* client, incl. private IPs) | App side: env `INSTANA_EXTRA_HTTP_HEADERS=x-forwarded-for;x-real-ip`. Or agent-wide: `com.instana.tracing.extra-http-headers: ['X-Forwarded-For', 'X-Real-IP']`. | The header appears as a **tag on the HTTP entry call** (e.g. `X-Forwarded-For = 192.168.178.130`), filterable/groupable in Analyze → Calls. |
| **Connection client IP on the Apache span** | The `mod_remoteip` config above (so Apache's `REMOTE_ADDR` is the real client). | The Apache/httpd call's client IP is the real client (depends on the httpd sensor reading `REMOTE_ADDR`). |

The header-capture mechanism is the more robust of the two and is the one that
does not depend on the `InternalProxy`/`TrustedProxy` private-IP subtlety —
configure both for completeness.

### Searching for the client IP in Instana (verified in the UI)

1. **Analyze → Calls**, set the time range.
2. **Add filter** → type **`header`** → pick **HTTP → Call Http Header**
   (the captured-headers tag; underlying key `call.http.header`). NB: there is
   no top-level "X-Forwarded-For" tag — searching "forwarded" finds nothing; the
   header lives inside the key/value `Call Http Header` tag.
3. Fill **Key = `x-forwarded-for`**, operator **equals**, **Value = `192.168.178.130`**.
4. The call list filters to matching requests (here: the `helloworld` service).
   Open a call → right-hand **Details** panel lists the captured headers:
   `Header x-forwarded-for  192.168.178.130`, `Header x-real-ip 192.168.178.130`,
   `Host 192.168.178.216`.
5. To see **all client IPs at once**: use **Add group → Call Http Header**
   (key `x-forwarded-for`) → one row per client IP with call counts/latency.

> **Agent-level header capture is global.** Setting `extra-http-headers` on the
> Instana agent affects all monitored services on that agent. The per-service
> env var (`INSTANA_EXTRA_HTTP_HEADERS`) is the contained option.

---

## 6. Quick verification

```bash
# Drive the full chain (front door = the LoadBalancer external IP)
curl -s http://192.168.178.216/hello | grep -E 'clientIp|xForwardedFor'
#   "xForwardedFor": "192.168.178.130"
#   "clientIp": "192.168.178.130"

# Apache resolved the real client?
kubectl -n xff-test logs deploy/apache -c apache | tail
#   client=192.168.178.130  xff="-"  "GET /hello HTTP/1.0"  200
```

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Apache logs `client=<proxy-pod-ip>`, `xff="<client>"` | `mod_remoteip` did not rewrite | Check the module is loaded; the peer must match an `Internal/TrustedProxy` entry |
| Public client works, private client does not | Using `RemoteIPTrustedProxy` | Switch to `RemoteIPInternalProxy` |
| Client IP disappears entirely | You trusted the client's own subnet | Trust only the proxy hop (pod CIDR), not the client range |
| App sees `X-Forwarded-For: <client>, <proxy>` (extra hops) | A proxy in the chain is not trusted, so it is not stripped | Add that proxy's IP/CIDR to the trusted list; the **first** entry is still the client |
| No header tag in Instana | Header capture not configured | Set `INSTANA_EXTRA_HTTP_HEADERS` (app) or `extra-http-headers` (agent) |

---

*Companion files: `20-apache.yaml` (this config, deployed), `README.md` (full
lab setup and architecture).*
