# Client IP in IBM ELM traces — Apache (IHS) X-Forwarded-For setup

**Why:** when a user reports a problem in ELM (jts / ccm / qm / rm / gc), you want
to find *their* requests in Instana by client IP — to see their errors, slow
calls and traces. By default the ELM app servers (WebSphere Liberty/WAS) see the
**reverse proxy** as the caller, not the user. The fix is to make the ELM reverse
proxy put the real client IP into **`X-Forwarded-For`**, and have Instana capture
it onto the trace.

These directives go into the **existing ELM reverse-proxy** `httpd.conf` (IBM HTTP
Server or Apache — both ship `mod_remoteip`, `mod_proxy`, `mod_headers`). You do
**not** add a new proxy; your current `ProxyPass`/plugin rules stay unchanged.

---

## Where the client IP must end up

```
user ─► [corporate LB?] ─► ELM reverse proxy (IHS/Apache) ─► Liberty/WAS (jts,ccm,…)
                                                              └─ Instana traces this entry
```
The Liberty/WAS HTTP entry call must carry `X-Forwarded-For: <user-ip>`. Pick the
section below that matches whether a load balancer sits in front of the proxy.

---

## A) ELM reverse proxy is **behind a load balancer** (LB sets X-Forwarded-For)

The LB terminates the user's connection, so the proxy sees the LB as the client.
Use `mod_remoteip` so the proxy recovers the real user IP and forwards it on.

```apache
LoadModule remoteip_module modules/mod_remoteip.so

RemoteIPHeader        X-Forwarded-For
RemoteIPInternalProxy 10.0.0.0/8        # <-- the LB IP/CIDR in front of the proxy

# (optional) prove it in the proxy access log; %a = corrected client IP
LogFormat "%a %t \"%r\" %>s xff=\"%{X-Forwarded-For}i\"" elmxff
CustomLog logs/access_log elmxff
```
- mod_proxy then forwards the corrected client to Liberty in `X-Forwarded-For`.
- **Use `RemoteIPInternalProxy` (not `RemoteIPTrustedProxy`)** — ELM users are
  usually on **private** corporate/VPN IPs (`10.x` / `192.168.x`), and
  `TrustedProxy` refuses to promote a private address to "client".
- **Trust only the LB**, never the users' own subnet, or the proxy strips the
  user IP as if it were a proxy.

---

## B) ELM reverse proxy is the **edge** (users hit IHS/Apache directly, no LB)

No inbound `X-Forwarded-For`, so **`mod_remoteip` is not needed** — `mod_proxy`
adds `X-Forwarded-For = the real client` on its own. Also strip any header a
client might forge.

```apache
LoadModule headers_module modules/mod_headers.so

RequestHeader unset X-Forwarded-For early   # anti-spoof: drop client-supplied XFF
# ProxyAddHeaders On  (default) -> mod_proxy adds X-Forwarded-For = client to ELM
```

---

## C) If the proxy routes via the **WebSphere plugin** (`plugin-cfg.xml`)

Many ELM reverse proxies forward to the app servers with the **WebSphere WebServer
Plugin** (`mod_was_ap24_http.so` + `plugin-cfg.xml`) instead of `mod_proxy`. Two
things differ:

1. The plugin passes the client IP to WAS/Liberty in its **private header `$WSRA`**
   (that is what `request.getRemoteAddr()` returns in ELM). Liberty **consumes**
   the `$WS*` headers, so they are usually **not** visible to Instana.
2. **The plugin does NOT add `X-Forwarded-For`** (unlike `mod_proxy`). You must set
   it yourself with `mod_headers` so Instana can capture it.

Requires **IHS 9 / Apache 2.4** (for `mod_remoteip` and `expr=`).

**Behind a load balancer:**
```apache
LoadModule remoteip_module modules/mod_remoteip.so
LoadModule headers_module  modules/mod_headers.so

RemoteIPHeader        X-Forwarded-For
RemoteIPInternalProxy 10.0.0.0/8     # the LB in front; corrects REMOTE_ADDR to the user

# Plugin won't forward XFF -> publish the corrected client IP as a header for Instana.
# 'set' overwrites any inbound value, so the client cannot spoof it.
RequestHeader set X-Forwarded-For "expr=%{REMOTE_ADDR}"
```

**Edge (no LB):**
```apache
LoadModule headers_module modules/mod_headers.so
# REMOTE_ADDR is already the real client; publish it as XFF for Instana (overwrite = spoof-safe).
RequestHeader set X-Forwarded-For "expr=%{REMOTE_ADDR}"
```

`$WSRA` keeps ELM's own `getRemoteAddr()` correct; you capture the
`X-Forwarded-For` you just set for the **trace tag** (since Liberty strips `$WS*`).

> IHS 8.5.5 (Apache 2.2) has no `mod_remoteip`/`expr=` — handle XFF on an upstream
> Apache/LB, or move to IHS 9.

---

## Make Instana show the IP on ELM traces

Capture the header on the Instana **agent** monitoring the ELM (Liberty/WAS)
hosts — `configuration.yaml`:

```yaml
com.instana.tracing:
  extra-http-headers:
    - 'X-Forwarded-For'
    - 'X-Real-IP'
```
Restart/let the agent reload. The user IP now appears on every ELM HTTP call as
the tag **`Call Http Header`** (key `x-forwarded-for`).

---

## Debug workflow: find one user's requests by IP

1. **Analyze → Calls**, set the time window of the incident.
2. **Add filter** → search `header` → **HTTP ▸ Call Http Header** →
   `key = x-forwarded-for`, **`contains`**, `value = <user-ip>`.
   *(Use **contains**, not equals — XFF can be a list; the user IP is the first entry.)*
3. Add `Service = <jts|ccm|qm|rm>` and/or toggle **Erroneous** to isolate their
   failures; open a call to read its full trace, errors and downstream calls.

**Recurring / VIP user → standing view (optional):**
Applications → **Add** → **Request attributes** → **HTTP Header** →
`x-forwarded-for` **contains** `<user-ip>` → name `ELM user <ip>`. This gives a
full APM perspective (calls, latency, errors, Smart Alerts) scoped to that user.
A custom-dashboard widget works the same way: data source **Applications**,
metric **Calls/Latency/Erroneous**, filter `Call Http Header x-forwarded-for
contains <user-ip>`.

---

### Verify

```bash
curl -s https://<elm-host>/jts/ -o /dev/null
# proxy access_log:  <user-ip> ... xff="<user-ip>"
```
In Instana, the call's **Details** panel shows `Header x-forwarded-for = <user-ip>`,
`Host = <elm-host>`. If it still shows the proxy/LB IP, re-check the trusted-proxy
range (A) or that XFF isn't being stripped (B).
