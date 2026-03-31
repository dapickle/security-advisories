# 🔒 Security Advisories Repository

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

This repository serves as a centralized archive for security advisories, vulnerability findings, and related research on software and applications. It provides a structured way to document, track, and share security issues discovered during research and assessments.

## 📋 Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Usage](#usage)
- [License](#license)

## 📖 Overview

The repository includes categorized security findings to ensure clarity and organization:

- **Public CVEs**: Vulnerabilities assigned a CVE ID and publicly disclosed.
- **Disputed Findings**: Issues that the client or vendor does not recognize as vulnerabilities or has declined to address.
- **NoReply Findings**: Vulnerabilities awaiting client response.

Each advisory maintains a historical record of security research, clearly marking the status and resolution of each finding.

## 📁 Repository Structure

```
security-advisories/
├── README.md                 # This file
├── CVEs/                     # Public CVE advisories
│   ├── [CVE-2026-XXXXX] Title.md
│   └── ...
├── Disputed/                 # Disputed findings
│   └── ...
└── NoReply/                  # Unresponded findings
    └── ...
```

- **[CVEs/](CVEs/)**: Contains detailed markdown files for each public CVE, following a standardized format.
- **[Disputed/](Disputed/)**: Advisories for findings disputed by the vendor or client.
- **[NoReply/](NoReply/)**: Findings pending client response.

## 🚀 Usage

1. **Browse Advisories**: Navigate to the relevant folder and open the markdown file.
2. **Search**: Use GitHub's search to find specific CVEs or keywords.
3. **Reference**: Cite advisories in reports or discussions using the CVE ID.



## 📜 License

This repository is licensed under the MIT License. See [LICENSE](LICENSE) for more details.

---

