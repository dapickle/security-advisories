## Title
Stored Cross-Site Scripting (XSS) via Response Rendering

### Summary
The chat interface renders HTML content from the LLM output without proper sanitization, allowing an attacker to inject malicious scripts. This enables execution of arbitrary JavaScript in the context of other users' browsers.

### Details
- Endpoint / component affected: Chat message rendering in the web UI  
- The system allows LLM-generated content to be **rendered as HTML**.  
- No sanitization or escaping is performed before rendering.  
- By injecting a payload such as `<svg><script>alert('XSS')</script></svg>` the script executes in the browser of the user viewing the response.  
- This could be exploited via **prompt injection**, user-supplied messages, or crafted API responses.

**Implications:**
- Theft of session cookies / tokens  
- Account takeover  
- Execution of actions on behalf of victims  
- Potential combination with other vulnerabilities (IDOR, session persistence, etc.)

### PoC
1. Configure a new empty flow. 
2. Set the flow like the following picture:
<img width="1043" height="545" alt="image" src="https://github.com/user-attachments/assets/03452135-dce8-425b-b9f9-006af47b6953" />

3. Click on Playground and send a message containing `<svg><script>alert('XSS')</script></svg>`
4. Check that the xss is executed and it is stored. When the user tries to reload the page the same xss persists. 

<img width="1345" height="805" alt="image" src="https://github.com/user-attachments/assets/8ccfd70f-59c2-4cda-8ddb-5a0140361a9d" />

**Note:** In the PoC, the AI component was simulated: the scenario was programmed to echo the input directly to produce the response. This was done to demonstrate the vulnerability more clearly. In a real scenario, the result would be identical if the LLM output contained the same payload.

### Impact
This is a **High severity Stored XSS** (depending on whether other users see the output).  

- Exploitable by attackers via crafted input to the LLM interface  
- Can compromise **confidentiality, integrity, and availability** of user accounts  
- Enables **remote code execution in the browser context**

**Impacted:** All users of the chat interface