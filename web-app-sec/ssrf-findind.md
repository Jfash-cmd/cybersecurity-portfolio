# SSRF Findings

##  Server-Side Request Forgery, Internal Admin Access via Stock Checker

### Summary

The application's stock checking feature accepts a full URL in the stockApi parameter and makes a server-side request to it without proper validation. This allows an attacker to make the server request arbitrary URLs, including internal services and the localhost admin interface.

### Severity

High. The vulnerability allows unauthenticated access to internal admin functionality, bypassing authentication controls by appearing to originate from a trusted location (localhost).

### Steps to Reproduce

1. Browse to any product page and click "Check stock"
2. Intercept the request in Burp Suite and send to Repeater
3. Observe the stockApi parameter contains a full URL
4. Modify the stockApi parameter to http://127.0.0.1/ and observe the server makes a request to localhost
5. Modify the stockApi parameter to http://127.0.0.1/admin and observe the admin interface is returned

### Proof of Concept

POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded

stockApi=http://127.0.0.1/admin

### Root Cause

The server fetches the URL supplied in the stockApi parameter without validating whether the destination is safe or permitted. No allowlist or filtering is applied to prevent requests to internal addresses.

### Remediation

Implement strict URL validation using an allowlist of permitted destinations. Parse the URL and validate the hostname against approved domains. Do not rely on string matching or pattern blocking.

## Server-Side Request Forgery, Filter Bypass via IP Shorthand

### Summary

The application blocks requests containing /admin in the URL. This filter can be bypassed using double URL encoding, which the filter fails to decode but the server processes correctly.

### Severity

Medium. The bypass allows an attacker to access blocked endpoints by obfuscating the URL path.

### Steps to Reproduce

1. Send a request with stockApi=http://127.1/admin and observe it is blocked
2. Send a request with stockApi=http://127.1/%2561dmin and observe the request succeeds
3. The filter sees %2561 and does not recognize it as "admin"
4. The server decodes %2561 twice: first to %61, then to a, resulting in a request to /admin

### Proof of Concept

POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded

stockApi=http://127.1/%2561dmin

### Root Cause

The filter checks the raw, undecoded input for blocked strings. The server decodes the input multiple times, creating a discrepancy between what the filter sees and what the server actually requests.

### Remediation

Apply the same decoding logic in the filter as the server uses. Ideally, decode the input fully before applying any checks, or perform validation on the final parsed URL rather than the raw string.

## Server-Side Request Forgery, Chained with Open Redirect

### Summary

The application strictly validates the stockApi parameter to only allow URLs on the same domain. However, the application contains an open redirect vulnerability in the /product/nextProduct endpoint that can be leveraged to bypass this restriction and access arbitrary internal systems.

### Severity

High. The chained vulnerability allows an attacker to bypass domain restrictions and access internal systems that would otherwise be unreachable, including internal admin interfaces.

### Steps to Reproduce

1. Confirm the open redirect: send a request to /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin and observe a 302 redirect to the supplied URL
2. In the POST /product/stock request, set stockApi=/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin
3. The server validates the URL (passes, as it starts with an allowed path), requests it, follows the redirect, and returns the admin interface
4. To delete a user, set stockApi=/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin/delete?username=carlos

### Proof of Concept

POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded

stockApi=/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin/delete?username=carlos

### Root Cause

Two vulnerabilities combine: (1) the open redirect allows arbitrary redirection to any URL, and (2) the SSRF filter only validates the initial URL, not the final destination after redirects. The server's HTTP client automatically follows redirects without re-validating each hop.

### Remediation

1. Fix the open redirect by validating the path parameter against an allowlist of permitted destinations
2. Disable automatic redirect following in the HTTP client, or validate each redirect hop against the same allowlist as the initial request

### References

1. https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/
2. https://portswigger.net/web-security/ssrf


   
