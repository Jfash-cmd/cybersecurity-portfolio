# CSRF Findings

### Summary
Juice Shop's profile page allows a logged in user to change their display name via a POST request to `/profile`. This endpoint includes no CSRF token at all, relying solely on the session cookie to identify the user, a textbook CSRF vulnerability at the application level.

### Severity
Medium. The action itself only changes a display name, not a highly sensitive value, but the complete absence of any CSRF protection on a state changing endpoint is a genuine, exploitable flaw in principle.

### Steps to Reproduce
1. Log into Juice Shop and capture the request sent when changing the username via `/profile`, using Burp Suite
2. Observe the request body contains only `username=value`, with no token parameter present anywhere
3. Build a hidden form targeting this endpoint, hosted on a separate origin (htmledit.squarefree.com)
4. Load the page containing this form while logged into Juice Shop in the same browser

### Proof of Concept
```
http://localhost:3000/profile
```
```html
<form action="http://localhost:3000/profile" method="POST">
    <input name="username" value="CSRF"/>
    <input type="submit"/>
</form>
<script>
    document.forms[0].submit();
</script>
```
### Root Cause
The server trusts the request purely based on the session cookie, with no secondary, unpredictable value (a CSRF token) required to confirm the request genuinely originated from the application's own pages.

### Remediation
Implement a server generated CSRF token, tied to the user's own session, required on every state changing request including this one.

### Note on exploitability in modern browsers
Testing confirmed the exploit above does not succeed on current versions of Chrome, due to SameSite Lax cookie restrictions, enforced by default since 2021. Since the attack requires a cross site POST request, and Lax only attaches cookies to cross site requests that are both GET and the result of genuine top level navigation, the session cookie is withheld, and the forged request fails silently. A GET based bypass was also tested, using `document.location` to satisfy Lax's conditions, but the endpoint does not process username changes via GET, only POST, so this bypass does not apply here either. This finding demonstrates a real application level vulnerability (missing CSRF token) that is nonetheless mitigated in practice by a separate, browser level defense on modern, default configured browsers.
