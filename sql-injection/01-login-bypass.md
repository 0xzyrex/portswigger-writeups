# Lab 01 — SQL Injection: Login Bypass

**Category:** SQL Injection  
**Difficulty:** 🟢 Apprentice  
**Platform:** PortSwigger Web Security Academy  
**Lab Link:** https://portswigger.net/web-security/sql-injection/lab-login-bypass  
**Completed:** 2026-07-26  
**Author:** [0xZyrexSec](https://github.com/0xzyrex)

 Objective

Log in to the application as the `administrator` user without knowing the password, by exploiting a SQL injection vulnerability in the login form.


 What Is This Vulnerability?

SQL injection happens when user-supplied input is inserted directly into a database query without sanitization. The database cannot distinguish between the developer's intended SQL and an attacker's injected code.

In a login form, the backend typically runs a query like this:

sql
SELECT * FROM users WHERE username = 'INPUT' AND password = 'INPUT'

If the application trusts the username field without sanitizing it, an attacker can break out of the string and modify the entire query — including commenting out the password check entirely.

 Recon — Understanding the Attack Surface

Opening the lab presents a standard e-commerce application with a login form at `/login`.

The form has two fields:
- `username`
- `password`

There is no visible CAPTCHA, rate limiting, or multi-factor authentication. Both fields are submitted via a standard POST request.

**The suspected backend query:**
```sql
SELECT * FROM users 
WHERE username = '[username input]' 
AND password = '[password input]'
```

The vulnerability theory: if the `username` field is injectable, I can inject a comment sequence (`--`) that causes the database to ignore the `AND password = '...'` portion of the query entirely.

🧪 Methodology

 Step 1 — Confirm the Injection Point

I first tested whether the username field accepts special characters that affect query logic. A single quote `'` is the standard first probe — if the app returns a database error or behaves differently, the field is vulnerable.

Test input in username:
```
'
```

**Result:** The application returned a server error — confirming the input is being inserted directly into a SQL query without escaping.

Step 2 — Craft the Bypass Payload

Since the backend query uses single quotes around the username, I need to:
1. Close the string with `'`
2. Select the target user (`administrator`)
3. Comment out everything after (including the password check) with `--`

**Payload:**
```
administrator'--
```

**What this does to the backend query:**
```sql
-- BEFORE (normal login attempt):
SELECT * FROM users WHERE username = 'administrator' AND password = 'anything'

-- AFTER (with payload):
SELECT * FROM users WHERE username = 'administrator'--' AND password = 'anything'
                                                     ^^
                                                     Everything after -- is ignored by the database
```

The `--` is an SQL comment sequence. The database stops reading at that point. The password check never executes.

Step 3 — Submit the Payload

In the login form:

| Field | Value |
|---|---|
| Username | `administrator'--` |
| Password | `anything` (literally anything — it's ignored) |

Clicked **Log in**.

Step 4 — Result

The application authenticated me as `administrator` and redirected to the admin panel.

```
HTTP/2 302 Found
Location: /my-account
```

The lab banner confirmed: **"Congratulations, you solved the lab!"**

 Real-World Impact

If this were a production application:
- **Full account takeover** — any account can be accessed by injecting their username
- **No password required** — authentication is completely bypassed
- **Admin access** — attacker gains the highest privilege level in the application
- **Data breach** — admin panel typically exposes all user data, orders, and PII
- **Zero forensic trace** — no failed login attempts are logged because the query succeeds

**CVSS Estimate:** Critical (9.8)
- Attack Vector: Network
- Attack Complexity: Low
- Privileges Required: None
- User Interaction: None
- Impact: Complete authentication bypass

🛡️ Fix / Mitigation

The Correct Fix — Parameterized Queries

The developer must **never** insert user input directly into a SQL string. The solution is parameterized queries (also called prepared statements), where the query structure is defined first and user input is passed separately:

**Vulnerable code (Python example):**
```python
# NEVER DO THIS
query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
cursor.execute(query)
```

**Secure code:**
```python
 ALWAYS DO THIS
query = "SELECT * FROM users WHERE username = ? AND password = ?"
cursor.execute(query, (username, password))
```

With parameterized queries, the database treats the input as pure data — never as executable SQL code. Even if the attacker sends `administrator'--`, the database looks for a user literally named `administrator'--` and finds nothing.

Additional Defenses (Defense in Depth)
- **Input validation** — reject inputs containing `'`, `--`, `;`, `/*` at the application layer
- **Least privilege** — the database account used by the app should only have SELECT on necessary tables, never DDL or admin rights
- **WAF** — a Web Application Firewall can detect and block common SQLi patterns
- **Error handling** — never expose raw database errors to users; they reveal query structure to attackers

🧠 What I Learned

The `--` comment trick is the cleanest login bypass in SQL injection. What clicked for me: the database executes what it receives — it cannot tell the difference between a developer's intended query and an attacker's modified one. The only way to fix this is at the code level with parameterized queries, not by filtering inputs (filtering can always be bypassed). Every login form I encounter from now on, my first question is: *is this username field building a raw SQL string?*

🔗 References
- [PortSwigger: SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger: SQL Injection Login Bypass Theory](https://portswigger.net/web-security/sql-injection#retrieving-hidden-data)
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
- [OWASP: Query Parameterization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html)


*Part of my [PortSwigger Web Security Academy writeup series](https://github.com/0xzyrex/portswigger-writeups) | [0xZyrexSec on Hashnode](#) | [@0x_zyrexSec on X](https://twitter.com/0x_zyrexSec)*
