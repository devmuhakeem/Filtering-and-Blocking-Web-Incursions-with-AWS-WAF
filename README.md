# Filtering and Blocking Web Incursions with AWS WAF

A hands-on AWS Builder Lab where I attacked OWASP Juice Shop with OWASP ZAP (SQL injection, persisted XSS, and a privilege-escalation challenge), then defended it with AWS WAF managed rules and a custom rule to close the one gap they missed.

## Scenario
The target was OWASP Juice Shop, an intentionally vulnerable web application, served through CloudFront and also through an Application Load Balancer as **ProtectedJuiceShop**. My task was to compromise it using OWASP ZAP, then build a web ACL in AWS WAF to stop the same attacks, and finally write a custom rule for the one exploit that got through the managed rule groups.

## What I did

### 1. Set up the attack environment
- Connected to the PenTestingHost Windows instance via Systems Manager Fleet Manager RDP, authenticated with a key pair
- Configured OWASP ZAP as a man-in-the-middle proxy and launched Firefox through ZAP's Manual Explore feature, pointed at the Juice Shop URL

### 2. Reconnaissance and SQL injection
- Generated baseline traffic: searched the site, registered a user, logged in
- Modified the search request's query string to `/rest/products/search?q=' or 1=1 --` (a boolean-based SQLi payload) and replayed it with ZAP's Request Editor
- The resulting unhandled exception revealed the application was running on **SQLite**

### 3. Persisted (stored) XSS attack
- Modified the `/api/Users` registration request, setting the email field to `<iframe src="javascript:alert('xss')">`
- The server accepted it, writing the payload into the database as a persisted XSS attack that would execute in the browser of anyone viewing that user later
- Noticed the JSON response included fields never set in the request, including `role`, hinting at hidden attributes

### 4. Two challenges
- **Bypass the login page**: applied the same boolean SQLi technique to the login request to authenticate without valid credentials
- **Create an admin user**: guessed that `role` accepted values beyond `customer`, sent `"role": "admin"` in the registration payload, and received back a new account with admin privileges, a mass assignment vulnerability

### 5. Built a web ACL in AWS WAF
- Created **JuiceShopACL**, associated with ProtectedJuiceShop (behind an Application Load Balancer)
- Added two AWS managed rule groups: the **Core rule set** and the **SQL database** rule group
- ACL capacity: 900 of 5,000 WCUs

### 6. Replayed the attacks against the protected app
- Boolean SQLi (search and login): blocked, 403 Forbidden
- Stored XSS: blocked, 403 Forbidden
- Admin role escalation (`"role": "admin"`): **not blocked** — managed rules have no way to know this specific field/value combination is a privilege escalation for this application

### 7. Closed the gap with a custom rule
- Created **CustomSQLiRuleForJuiceShop**: matches all statements (AND) where the URI path contains `api/Users` AND the body contains `"role": "admin"` (oversize handling: Continue)
- Replayed the admin-creation payload: blocked, 403 Forbidden

## Key takeaways
- Attacking an application before defending it makes the defensive rule feel necessary rather than theoretical
- Verbose error messages are reconnaissance gold; this one revealed the database engine in a single request
- Unexpected fields in an API response (like `role`) are a signal worth investigating, not noise to ignore
- Managed rule groups catch broad, well-known attack signatures; an application-specific business logic flaw (mass assignment) needed a custom rule
- Narrow, AND-joined custom rules reduce false positives but cost more WCUs, especially with JSON body inspection
- A WAF rule protects the path it's attached to, not every path to the origin — defense in depth still matters

## Tools
OWASP ZAP, OWASP Juice Shop, AWS WAF & Shield, AWS Systems Manager Fleet Manager, Amazon CloudFront, Application Load Balancer

---
*Completed as an AWS Builder Lab (Filtering and Blocking Web Incursions with AWS WAF). Target and payloads are standard OWASP Juice Shop training exercises against an intentionally vulnerable app.*
