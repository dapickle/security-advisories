## Title
Session Persistence After Password Change

### Summary
After changing their password via `PATCH /api/v1/users/{user_id}/reset-password`, the application **does not invalidate existing sessions**. Any previously issued session cookies remain valid, allowing an attacker with a stolen session token to continue accessing the account even after the password is changed.

### Details
- Endpoint affected: `PATCH /api/v1/users/{user_id}/reset-password`  
- The password reset logic only updates the stored password hash.  
- No mechanism exists to invalidate active sessions tied to the account.  
- This breaks standard security expectations, as changing a password should terminate all active sessions except possibly the current one.

Potential consequences:
- Persistent account takeover if an attacker obtained the session cookie before the password change. 
- Compromised user sessions remain active indefinitely.

### PoC
1. Log in as a user and capture the session cookie or token.  
2. Change the password:
```
PATCH /api/v1/users/{user_id}/reset-password
Cookie: xxxx
{
"password": "test2"
}
```
3. Continue using the old session cookie or token to perform authenticated actions on the account.  
4. Observe that all requests succeed despite the password change.

### Impact
This is a **session persistence / weak session invalidation** issue.

- An attacker who has stolen a session token can **retain full access** to the account even after a password change.  
- Reduces effectiveness of password resets as a security mitigation.  
- Compromises **confidentiality, integrity, and availability** of the affected user account.

**Impacted:** All users of the application.