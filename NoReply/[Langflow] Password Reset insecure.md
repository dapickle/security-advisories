## Title
Password Reset Without Current Password Verification

### Summary
The endpoint `PATCH /api/v1/users/{user_id}/reset-password` does not require the current password to authorize a password change. Any attacker with access to a valid session can change the account password without verifying the user's identity, enabling account takeover persistence.

### Details
The password reset logic only checks that the new password differs from the existing hashed password, but does not require the user to provide their current password.

This weakens authentication guarantees for sensitive actions. If an attacker gains access to a valid session (e.g., via XSS, token leakage, or shared device), they can change the victim’s password without knowing the original one.

Affected code:
- `users.py:108-135`

### PoC
1. Authenticate as a valid user and obtain a session/token
2. Send the following request:
```
PATCH /api/v1/users/{user_id}/reset-password
Authorization: Bearer <valid_token>
Content-Type: application/json

{
"password": "NewSecurePassword123!"
}
```

3. Observe that:
   - The password is successfully changed
   - No current password is required

4. Attempt to log in with the old password → it fails  
5. Log in with the new password → it succeeds  

### Impact
This is a **missing re-authentication for sensitive action** issue.

An attacker with temporary access to a valid session can:
- Permanently take over the account by changing the password
- Lock out the legitimate user

Impacted:
- Any authenticated user account
- Systems relying on session security without additional verification layers

This issue becomes significantly more severe when combined with other vulnerabilities such as XSS or session token leakage.