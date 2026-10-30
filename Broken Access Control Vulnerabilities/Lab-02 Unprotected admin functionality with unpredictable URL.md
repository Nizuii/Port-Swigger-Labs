# Lab: Unprotected Admin Functionality with Unpredictable URL

| | |
|---|---|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | Access Control Vulnerabilities |
| **Difficulty** | Apprentice |
| **Vulnerability** | Vertical privilege escalation: unprotected admin functionality (URL disclosed in client-side code) |
| **Objective** | Access the admin panel and delete the user `carlos` |
| **Tools used** | Firefox, Browser Developer Tools (Debugger) |

---

## 1. Challenge Description

The lab description states:

> This lab has an unprotected admin panel. It's located at an unpredictable location, but the location is disclosed somewhere in the application.
>
> Solve the lab by accessing the admin panel, and using it to delete the user `carlos`.

<img width="1533" height="784" alt="image" src="https://github.com/user-attachments/assets/a36e740f-92dc-4bd7-9de6-0b962184918a" />

---

## 2. Recon

The lab opens on a standard shop front-end ("We Like To Shop") with a product catalogue, a **Home** link and a **My account** link. The description tells me two important things:

1. An admin panel exists and is **not protected**.
2. Its URL is **unpredictable**, so guessing paths like `/admin` is unlikely to work, but the location is **disclosed somewhere in the application**.

<img width="1533" height="784" alt="image" src="https://github.com/user-attachments/assets/7d824151-6c04-4654-aa8f-5810efa5ff7a" />

### Attempt 1: `robots.txt` (dead end)

Since `robots.txt` leaked the path in the previous lab, I checked it first:

```
https://<LAB-ID>.web-security-academy.net/robots.txt
```

The server returned `"Not Found"`, so there was nothing useful here.

<img width="1535" height="784" alt="image" src="https://github.com/user-attachments/assets/9e42601a-27a7-4357-adcf-8db073be4855" />

### Attempt 2: Client-side source code (success)

The description says the location is disclosed *somewhere in the application*. Applications often leak information in client-side code, so I opened the browser Developer Tools (**F12**) and went to the **Debugger** tab to read the page's source.

In the `(index)` document I found an inline JavaScript block near the bottom of the page:

```javascript
var isAdmin = false;
if (isAdmin) {
    var topLinksTag = document.getElementsByClassName("top-links")[0];
    var adminPanelTag = document.createElement('a');
    adminPanelTag.setAttribute('href', '/admin-spjtyk');
    adminPanelTag.innerText = 'Admin panel';
    topLinksTag.append(adminPanelTag);
    var pTag = document.createElement('p');
    pTag.innerText = '|';
    topLinksTag.appendChild(pTag);
}
```

<img width="1535" height="784" alt="image" src="https://github.com/user-attachments/assets/62e702f4-afd8-4643-b8ec-2782e6beffff" />

The script only adds the "Admin panel" link to the navigation when `isAdmin` is `true`. As a normal user, the link is never rendered on screen, but the admin URL, `/admin-spjtyk`, is still **shipped to every visitor** in the page source.

---

## 3. Vulnerability Analysis

There are two problems stacked on top of each other:

1. **Information disclosure:** The "hidden" admin URL is embedded in JavaScript sent to every user. Hiding a link in the UI with a client-side `if (isAdmin)` check does not hide the URL from anyone who reads the source.
2. **Missing server-side access control:** Even once I knew the URL, a properly secured application would reject my request. This one served the admin panel to an unauthenticated visitor.

The root cause is the same as the previous lab: the application relies on **security through obscurity**. An unpredictable URL only slows down guessing. It is not an access control, and it fails completely as soon as the URL leaks.

---

## 4. Exploitation

### Step 1: Browse to the discovered admin path

```
https://<LAB-ID>.web-security-academy.net/admin-spjtyk
```

The admin panel loaded without any login, role check or redirect, listing two users, each with a **Delete** link:

- `wiener`
- `carlos`

<img width="1536" height="784" alt="image" src="https://github.com/user-attachments/assets/9257837c-e2fb-4f32-8166-6d0486dddb5a" />

### Step 2: Delete the target user

I clicked **Delete** next to `carlos`. The application responded with **"User deleted successfully!"** and the user list now showed only `wiener`.

### Step 3: Confirm

The lab banner changed to **Solved** and displayed *"Congratulations, you solved the lab!"*

<img width="1530" height="784" alt="image" src="https://github.com/user-attachments/assets/46f5fe24-e4e1-4422-b327-a229c11dac04" />

---

## 5. Result

| Check | Status |
|---|---|
| `robots.txt` checked (nothing found) | Done |
| Admin URL `/admin-spjtyk` found in client-side JavaScript | Done |
| Panel accessed without authentication | Done |
| User `carlos` deleted | Done |
| Lab status | **Solved** |

---

## 6. Impact

In a real application, this flaw would let any anonymous visitor who reads the page source:

- Delete, modify or create user accounts
- Escalate privileges or reset other users' credentials
- Disrupt the service or destroy data
- Use the admin panel as a foothold for further attacks

An "unpredictable" URL gives a false sense of security: the barrier is a single right-click and **View Source**.

---

## 7. Remediation

1. **Enforce authentication and role-based authorisation on the server** for every admin endpoint. Deny by default and require an administrator role.
2. **Never rely on obscure URLs as a security control.** Assume every URL will eventually be discovered.
3. **Don't leak privileged paths in client-side code.** If a link is only for admins, render it server-side for admin sessions only, rather than shipping it to everyone and toggling it with JavaScript.
4. **Remove sensitive paths from public files** such as `robots.txt`, `sitemap.xml`, JavaScript bundles and HTML comments.
5. **Log and alert** on requests to administrative paths from non-admin sessions.

---

## 8. Key Takeaways

- When `robots.txt` fails, look at the **client-side code**: inline scripts, linked `.js` files, HTML comments and hidden form fields all commonly leak paths.
- A client-side check like `if (isAdmin)` is **UI logic, not security**. The data inside the branch is still delivered to the browser.
- Use the Developer Tools **Debugger/Sources** tab and **Search** (Ctrl+Shift+F) to grep for keywords like `admin`, `panel`, `api`, `href` across all loaded scripts.
- The same lesson as the previous lab applies: always test whether a discovered page actually enforces access control.

---

## 9. References

- [PortSwigger: Access control vulnerabilities](https://portswigger.net/web-security/access-control)
- [PortSwigger: Unprotected functionality](https://portswigger.net/web-security/access-control#unprotected-functionality)
