### Summary
An SSRF vulnerability exists in n8n due to the SSRF protection mechanism being bypassed entirely in the OAuth2 dynamic client registration flow. An authenticated attacker can force the server to make arbitrary HTTP requests to internal services or cloud metadata endpoints, potentially leading to sensitive data exposure such as IAM credentials.

### Details
The application exposes two key issues:

1. **SSRF Protection Disabled by Default**
   The SSRF protection feature controlled by `N8N_SSRF_PROTECTION_ENABLED` defaults to `false`, leaving deployments unprotected unless explicitly configured.

   - File: `ssrf-protection.config.ts:55-56`

2. **Bypass in OAuth2 Discovery Flow**
   The OAuth2 dynamic discovery flow implemented in:
   - `OauthService.generateAOauth2AuthUri()` (`oauth.service.ts:395-412`)

   makes outbound HTTP requests using `axios.get()` directly, bypassing the SSRF protection layer.

   The only validation applied is via:
   - `validateOAuthUrl()` (`validate-oauth-url.ts:1-28`)

   This validation only enforces that the URL uses `http://` or `https://` schemes, but does not protect against:
   - Private IP ranges (e.g., 10.0.0.0/8, 192.168.0.0/16)
   - Loopback addresses (127.0.0.1)
   - Link-local addresses (169.254.169.254)
   - DNS resolution to internal hosts

As a result, user-controlled input (`serverUrl`) is used in server-side HTTP requests without sufficient validation or routing through the SSRF protection mechanism.

### PoC
**Step 0 – Enable SSRF protection**

```
export N8N_SSRF_PROTECTION_ENABLED=true  
```

**Step 1 – Create OAuth2 Credential**
```http
POST /rest/credentials
Content-Type: application/json

{
  "name": "ssrf-test",
  "type": "oAuth2Api",
  "data": {
    "useDynamicClientRegistration": true,
    "serverUrl": "http://[malicious_server]"
  }
}
```

**Step 2 – Trigger OAuth Flow**
```
GET /rest/oauth2-credential/auth?id=[cred_id]&useDynamicClientRegistration=true&serverUrl=[malicious_server]&ignoreSSLIssues=true&grantType=authorizationCode&authUrl=&accessTokenUrl=&clientId=&clientSecret=&scope=&authQueryParameters=&authentication=header&sendAdditionalBodyProperties=false&additionalBodyProperties=&tokenExpiredStatusCode=401&allowedHttpRequestDomains=all&allowedDomains= 
```


### Impact
- **Vulnerability Type:** Server-Side Request Forgery (SSRF)  
- **Attack Vector:** Authenticated user  
- **Affected Components:** OAuth2 dynamic client registration flow  

An attacker can:
- Bypass SSRF protection
- Access internal-only services (localhost, private IP ranges)
- Interact with internal APIs and administrative interfaces
- Potentially exfiltrate sensitive data such as IAM credentials
- Use the server as a pivot to scan and enumerate internal networks

