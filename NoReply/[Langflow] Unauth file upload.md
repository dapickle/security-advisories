## Title
Unauthenticated Deprecated file upload endpoint 

### Summary
The deprecated POST `/api/v1/upload/{flow_id}` endpoint in endpoints.py has no authentication dependency — `no dependencies=[Depends(...)]` in its decorator and no `current_user` parameter. Any unauthenticated actor can upload arbitrary files into the server's cache directory under any flow UUID they specify. endpoints.py:974-999

### Details
The storage path itself uses an sha256 hash of the file content as the filename, but there is no ownership check and no authentication.

### PoC
1. Send a post request to:
 ```
POST --> /api/v1/upload/{flow-id}
Attach test file
```
<img width="1578" height="487" alt="image" src="https://github.com/user-attachments/assets/2e76d6a7-d4a1-4ca7-ad6b-f8650e6269fe" />

2. Check that unauthenticated users can upload files to the server. Besides the response disclosures internal absolute path of the server. 

### Impact
This is an **unauthenticated arbitrary file upload vulnerability**.

An attacker can upload files to the server under any `flow_id` without authentication, leading to:
- Potential **denial of service** (disk exhaustion)
- **Data poisoning or overwrite** of cached flow files
- Upload of **malicious files** that may be processed later

**Impacted:**
- The application server and its storage
- All users relying on affected flows