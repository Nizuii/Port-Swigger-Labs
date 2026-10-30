# Lab: User Role Controlled by Request Parameter

| | |
|---|---|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | Access Control Vulnerabilities |
| **Difficulty** | Apprentice |
| **Vulnerability** | Vertical privilege escalation: user role controlled by a forgeable attribute (cookie) |
| **Objective** | Access the admin panel and delete the user `carlos` |
| **Tools used** | Firefox, Burp Suite (Proxy, Repeater) |
| **Credentials provided** | `wiener:peter` |

---

## 1. Challenge Description

The lab description states:

> This lab has an admin panel at `/admin`, which identifies administrators using a forgeable cookie.
>
> Solve the lab by accessing the admin panel and using it to delete the user `carlos`.
>
> You can log in to your own account using the following credentials: `wiener:peter`

<img width="1529" height="784" alt="image" src="https://github.com/user-attachments/assets/46d34491-1db5-40fd-b491-fbb93d498c49" />

---

## 2. Recon

The lab opens on a standard shop front-end ("We Like To Shop"). This lab is different from the previous two: it doesn't ask me to find a hidden admin URL, it tells me the path outright (`/admin`) and states that administrator status is decided by a **forgeable cookie**. The task is to find that cookie and forge it.

<img width="1530" height="784" alt="image" src="https://github.com/user-attachments/assets/a787fc14-4f69-41ed-8d13-3b51c0d8f0c1" />

### Step 1: Try the admin panel unauthenticated

I first browsed directly to:

```
https://<LAB-ID>.web-security-academy.net/admin
```

The server responded with:

```
Admin interface only available if logged in as an administrator
```

So, unlike the first two labs, this admin panel **does** check something before granting access.

<img width="1530" height="784" alt="image" src="https://github.com/user-attachments/assets/69541599-7a01-40fb-8c41-688422f5e8a4" />

### Step 2: Log in as the provided low-privilege user

I logged in with the supplied credentials `wiener:peter` and confirmed I was authenticated via the **My account** page (`/my-account?id=wiener`).

![Uploading image.png…]()

I tried `/admin` again as `wiener`. The result was the same rejection message, confirming `wiener` is a normal, non-admin user.

---

## 3. Vulnerability Analysis

To understand *how* `/admin` decides who is an administrator, I opened Burp Suite and inspected the proxied request in **HTTP history**.

The request to `/admin` carried this header:

```
Cookie: session=0wgz7QcCpqzZySxrORDTtfeacPXrERc9; Admin=false
```

![Burp HTTP history: GET /admin with cookie Admin=false, response says "only available if logged in as administrator"](images/05-burp-admin-false.png)

This is the vulnerability: **the server trusts a client-supplied cookie (`Admin=false` / `Admin=true`) to decide authorization**, instead of checking the user's actual role against a server-side record (e.g. a database role tied to the session). Since cookies are fully controlled by the client, this value can simply be edited and replayed.

> **Note:** My original testing notes called this a "GET request parameter," but as the Burp capture shows, `Admin` is actually a **separate cookie** sent alongside `session`, not a URL or body parameter. I've corrected that distinction in this report, since it matters for the root-cause explanation.

---

## 4. Exploitation

### Step 1: Send the request to Repeater and flip the cookie

I sent the `/admin` request to **Burp Repeater**, changed the cookie from:

```
Admin=false
```

to:

```
Admin=true
```

and resent the request.

### Step 2: Confirm admin access

The response now rendered the full admin interface, listing both users with **Delete** links:

```html
<a href="/admin">Admin panel</a>
...
<h1>Users</h1>
wiener - <a href="/admin/delete?username=wiener">Delete</a>
carlos - <a href="/admin/delete?username=carlos">Delete</a>
```

![Burp Repeater: Admin=true cookie returns the full admin panel with delete links](images/06-burp-admin-true-panel.png)

This confirmed the forged cookie was sufficient to bypass the access control check entirely.

### Step 3: Delete the target user

I changed the request line in Repeater to the delete endpoint shown in the response, keeping the forged `Admin=true` cookie:

```
GET /admin/delete?username=carlos HTTP/2
Host: <LAB-ID>.web-security-academy.net
Cookie: session=0wgz7QcCpqzZySxrORDTtfeacPXrERc9; Admin=true
```

The server responded:

```
HTTP/2 302 Found
Location: /admin
```

A `302` redirect back to `/admin` confirmed the deletion request was accepted and processed.

![Burp Repeater: delete request for carlos returns 302 Found](images/07-burp-delete-carlos.png)

### Step 4: Confirm in the browser

I navigated back to `/admin` in Firefox (with the browser's own session, after the lab's state updated) and the lab banner changed to **Solved**, with the confirmation banner *"Congratulations, you solved the lab!"*

![Lab solved banner](images/08-lab-solved.png)

---

## 5. Result

| Check | Status |
|---|---|
| `/admin` rejects unauthenticated access | Confirmed |
| `/admin` rejects `wiener` (non-admin) | Confirmed |
| Role-determining cookie (`Admin=false`) identified via Burp | Done |
| Cookie forged to `Admin=true` | Done |
| Full admin panel accessed | Done |
| User `carlos` deleted via `/admin/delete?username=carlos` | Done |
| Lab status | **Solved** |

---

## 6. Impact

Because the user's privilege level is derived entirely from a value the client controls:

- Any authenticated (or even unauthenticated, depending on implementation) user can escalate to administrator by setting one cookie.
- An attacker gains full administrative control: deleting accounts, and in a real system potentially modifying data, viewing sensitive records, or changing application configuration.
- This is a textbook example of **broken access control via client-side trust**, explicitly listed in the OWASP Top 10 (A01:2021 – Broken Access Control).

---

## 7. Remediation

1. **Never determine authorization from client-supplied data** (cookies, hidden fields, query parameters, headers). Roles must be looked up server-side from a trusted store (session data tied to a database record), not read back from values the client sent.
2. **Store role/permission state server-side**, keyed to the authenticated session ID, and re-validate it on every privileged request.
3. **Sign or encrypt** any session-related data that must round-trip through the client, and verify its integrity server-side, if role data absolutely must travel via cookie (not recommended for authorization decisions).
4. **Apply the principle of least privilege** and centralize access-control checks in one place (middleware/framework-level authorization) rather than scattering logic that trusts client input.
5. **Test privilege boundaries** for every role using tools like Burp Repeater: attempt to tamper with any value that looks like it encodes a role or permission.

---

## 8. Key Takeaways

- Always inspect **cookies**, not just URL/body parameters, when hunting for access-control flaws — Burp's HTTP history makes this easy to spot.
- A rejection message like *"only available if logged in as an administrator"* confirms a check exists; the next question is always **what decides the outcome of that check, and who controls it**.
- Burp Repeater is the fastest way to tamper with a single request value (cookie, parameter, header) and observe the server's reaction without replaying an entire browser flow.
- A `302 Found` on an action endpoint (like `/admin/delete`) is a useful signal that the server accepted and processed the request, even without a visible success message.

---

## 9. References

- [PortSwigger: Access control vulnerabilities](https://portswigger.net/web-security/access-control)
- [PortSwigger: User role controlled by user-controllable parameter](https://portswigger.net/web-security/access-control#user-role-controlled-by-user-controllable-parameter)
- [OWASP Top 10: A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
