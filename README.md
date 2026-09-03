# CTF Writeup: InvoiceVault
**Platform:** WebverseLabs Pro  
**Difficulty:** Easy  
**Category:** IDOR (Insecure Direct Object Reference)  
**Flag:** `WEBVERSE{7e630e8fbeb260c33005125bbf594bfc}`

---

## Briefing Summary

InvoiceVault is a six-person SaaS platform serving around four thousand freelancer accounts. A new account-export endpoint was scaffolded from an internal admin tool during a rushed week before a cust...

---

## Reconnaissance

After registering an account and logging in, the **Settings** page revealed an "Export account data" feature — a button that downloads a ZIP archive containing `invoices.csv` and `account_info.txt`.

![Account Settings page showing Export account data feature](image/Screenshot%202026-09-03%20092329.png)

This immediately stood out. An export feature that packages account data server-side is a strong candidate for IDOR — especially one that was "scaffolded from an internal admin tool."

---

## Vulnerability Analysis

**IDOR — Insecure Direct Object Reference (OWASP A01:2021)**

The export endpoint accepts a `user_id` parameter in the POST body:

```
POST /api/account/export
Content-Type: application/json

{"user_id": 3}
```

The server generates and returns a ZIP archive for the account matching that ID — without verifying that the requesting user owns that account. Any authenticated user can export any other user's dat...

The root cause: the endpoint was originally built as an internal admin tool where any user ID was valid. When it was repurposed for customer-facing data portability, the ownership check was never adde...

---

## Exploitation

### Step 1: Intercept the export request

With Burp Suite proxy running, clicking "Download export" captured the request:

```
POST /api/account/export HTTP/2
Host: 5a2fc908-4498-remittance-2a414.challenges.webverselabs-pro.com
Content-Type: application/json

{"user_id": 3}
```

My account was assigned `user_id: 3`.

### Step 2: Test IDOR by modifying user_id

The request was sent to Burp Repeater. Changing `user_id` to `1` returned a `200 OK` with a ZIP archive belonging to a different user — confirming the IDOR vulnerability.

![Burp Repeater showing user_id:1 returning another user's ZIP export](image/Screenshot%202026-09-03%20092551.png)

### Step 3: Enumerate to find the flag

Tested `user_id: 2` via Burp Intercept:

![Burp Intercept showing user_id:2 modification](image/Screenshot%202026-09-03%20092634.png)

The ZIP archive for `user_id: 2` contained `account_info.txt` with the flag embedded in the account data.

---

## Impact

Any authenticated InvoiceVault user can:

1. Export any other user's account data by iterating `user_id`
2. Access full invoice history, billing details, and account information for all 4,000+ accounts
3. Enumerate the entire user base sequentially

Since user IDs appear to be sequential integers starting from 1, the entire customer database is trivially enumerable with a simple loop.

---

## Remediation

1. **Enforce ownership server-side** — the export endpoint must verify that the authenticated user's ID matches the requested `user_id` before generating the export
2. **Never trust client-supplied IDs for ownership** — derive the user ID from the authenticated session, not from the request body
3. **Use UUIDs instead of sequential integers** — makes enumeration significantly harder, though not a substitute for proper authorization
4. **Audit all endpoints scaffolded from internal tools** — internal admin tools intentionally bypass access controls; those bypasses must be removed before customer-facing deployment
5. **Clear the backlog ticket** — the engineer who filed the follow-up ticket was right

---

*Writeup by Amit | WebverseLabs Pro CTF Series*
