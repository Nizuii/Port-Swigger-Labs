# Lab: Unprotected Admin Functionality

| | |
|---|---|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | Access Control Vulnerabilities |
| **Difficulty** | Apprentice |
| **Vulnerability** | Vertical privilege escalation: unprotected admin functionality |
| **Objective** | Delete the user `carlos` |
| **Tools used** | Firefox (no Burp required) |

---

## 1. Challenge Description

The lab description states:

> This lab has an unprotected admin panel. Solve the lab by deleting the user `carlos`.

<img width="1532" height="784" alt="image" src="https://github.com/user-attachments/assets/1c6b7d28-eb9e-4706-8b2e-a85d7d302a46" />

---

## 2. Recon

After launching the lab, I landed on a standard shop front-end ("We Like To Shop") with a product catalogue, a **Home** link and a **My account** link. Nothing on the page pointed to an admin area, so I started with the cheapest and most common recon step for a hidden path: checking `robots.txt`.

<img width="1532" height="784" alt="image" src="https://github.com/user-attachments/assets/1c67ab99-8aa7-45ec-aadf-b20d1a8c404e" />

### Finding the hidden path

I browsed to:

```
https://<LAB-ID>.web-security-academy.net/robots.txt
```

The file contained:

```
User-agent: *
Disallow: /administrator-panel
```

![robots.txt disclosing /administrator-panel](images/03-robots-txt.png)

`robots.txt` is meant to tell search-engine crawlers which paths not to index. It is **not** an access control mechanism, and anyone can read it. Developers often list sensitive paths in it, which ends up advertising exactly where an attacker should look.

---

## 3. Vulnerability Analysis

The `Disallow` entry told me that `/administrator-panel` exists. The real question was whether the server enforces any authentication or authorisation on it.

I browsed to `/administrator-panel` without logging in. The server returned the full admin interface with no login prompt, no role check and no redirect. This confirms the root cause:

- **No server-side access control** is applied to the admin functionality.
- The application relied on **security through obscurity**: the admin URL was "hidden" but not protected.

This is a **vertical privilege escalation** issue. An anonymous, unprivileged user can perform actions that should be restricted to administrators.

---

## 4. Exploitation

### Step 1: Access the admin panel directly

```
https://<LAB-ID>.web-security-academy.net/administrator-panel
```

The panel loaded without any authentication and listed the application's users, each with a **Delete** link:

- `wiener`
- `carlos`

![Admin panel accessible without authentication](images/04-admin-panel.png)

### Step 2: Delete the target user

I clicked **Delete** next to `carlos`. The application responded with **"User deleted successfully!"** and the user list now showed only `wiener`.

### Step 3: Confirm

The lab banner changed to **Solved** and displayed *"Congratulations, you solved the lab!"*

![Lab solved: carlos deleted](images/05-lab-solved.png)

---

## 5. Result

| Check | Status |
|---|---|
| Admin panel found via `robots.txt` | Done |
| Panel accessed without authentication | Done |
| User `carlos` deleted | Done |
| Lab status | **Solved** |

---

## 6. Impact

In a real application, this flaw would let any anonymous visitor:

- Delete, modify or create user accounts
- Take over the application by promoting their own account or resetting other users' credentials
- Disrupt the service or destroy data
- Use the admin panel as a foothold for further attacks, such as stored XSS or file upload through admin features

---

## 7. Remediation

1. **Enforce authentication and role-based authorisation on the server** for every admin endpoint. Deny by default and require an administrator role.
2. **Never rely on obscurity.** An unguessable or hidden URL is not an access control.
3. **Don't list sensitive paths in `robots.txt`.** It is public and acts as a roadmap for attackers. Protect the paths properly instead.
4. **Centralise access control** in middleware or a single enforcement layer, so new admin routes can't be added without protection.
5. **Log and alert** on access attempts to administrative paths by non-admin sessions.

---

## 8. Key Takeaways

- Always check `/robots.txt` (and `/sitemap.xml`) during recon. It often leaks hidden paths.
- "Hidden" is not the same as "protected". Always test whether a discovered page actually enforces access control.
- For access control labs, test every privileged function **as an unauthenticated user and as a low-privilege user**.
- Common admin paths worth guessing when `robots.txt` doesn't help: `/admin`, `/administrator`, `/admin-panel`, `/administrator-panel`.

---

## 9. References

- [PortSwigger: Access control vulnerabilities](https://portswigger.net/web-security/access-control)
- [PortSwigger: Vertical privilege escalation](https://portswigger.net/web-security/access-control#vertical-privilege-escalation)
