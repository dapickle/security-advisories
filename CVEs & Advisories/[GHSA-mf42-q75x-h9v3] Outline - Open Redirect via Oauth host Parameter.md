## Summary  
An Open Redirect vulnerability in the OAuth flow allows an attacker to control the post-authentication redirect destination by manipulating the `host` query parameter. This parameter is stored without validation and later used during error handling, enabling redirection to an attacker-controlled domain. This can be leveraged for phishing attacks against users whose authentication is expected to fail.

This only affects cloud-hosted deployments. Self-hosted instances use env.URL for the redirect, ignoring state.host entirely.

---

## Details  
The vulnerability originates from improper handling of the `host` query parameter during OAuth initiation:

- The parameter is accepted directly from user input:
  ```ts
  const host = ctx.query.host?.toString();
  ```
- It is stored in the OAuth `state` cookie without validation against trusted domains (e.g., team subdomains or registered custom domains).

During the OAuth callback flow:

- The `state` cookie is read to determine the redirect destination.
- Although `StateStore.verify` clears the cookie using:
  ```ts
  ctx.cookies.set(...)
  ```
  this only affects the response. The original cookie remains accessible via:
  ```ts
  ctx.cookies.get("state")
  ```
- If authentication fails and an error with `err.id` is triggered, the application constructs a redirect URL using the unvalidated `host` value.

This results in a redirect to an attacker-controlled domain.

**Key Issue:**  
No validation is performed to ensure that `host` belongs to a trusted domain before storing or using it.

---

## PoC  

### Steps to reproduce:

1. Craft a malicious OAuth initiation URL:
   ```
   https://app.getoutline.com/auth/google?host=attacker.com
   ```

2. Send the link to a victim.

3. Victim clicks the link:
   - A `state` cookie is set with:
     ```
     host=attacker.com
     ```

4. Victim is redirected to Google OAuth and proceeds with authentication.

5. After authentication:
   - Google redirects back to the application callback endpoint.
   - State verification succeeds.

6. Trigger an authentication failure:
   - Example: gmail-account-creation cannot create account using personal gmail address

7. Error handler executes:
   - Reads `host` from the request cookie (`attacker.com`).
   - Issues a redirect:
     ```
     302 → https://attacker.com/?notice=*****
     ```
<img width="1307" height="723" alt="image" src="https://github.com/user-attachments/assets/f7b6bea1-37f7-4605-acec-93883c4f4315" />

---

## Impact  

- **Vulnerability type:** Open Redirect  
- **Attack vector:** Social engineering / phishing  
- **Affected users:**  
  - Users whose authentication is expected to fail (e.g., unauthorized domains)

An attacker can redirect victims to a malicious site that mimics the legitimate login page, potentially leading to credential theft.

**Important note:**  
The redirect only occurs on authentication errors, making it a targeted attack vector rather than a universal one.

---

## Recommendation  

Implement strict validation for the `host` parameter before storing it in the `state` cookie:

- Allow only:
  - Registered team subdomains
  - Registered custom domains
  - The application's apex domain

- Reject or sanitize any untrusted values.

Additionally:

- Avoid relying on request cookies after they have been "cleared" in the response.
- Consider binding OAuth state strictly to server-side session data instead of client-controlled cookies.
