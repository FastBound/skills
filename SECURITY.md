# Security Policy

## Reporting a vulnerability

FastBound publishes a coordinated vulnerability disclosure policy and a
machine-readable `security.txt`. Please use those official channels:

- **Contact:** security@fastbound.com
- **Disclosure policy:** https://www.fastbound.com/security-vulnerability-disclosure-policy/
- **security.txt:** https://www.fastbound.com/.well-known/security.txt
- **PGP key:** https://www.fastbound.com/.well-known/security.asc

Please do not open a public GitHub issue for security reports — email
security@fastbound.com instead so the issue can be handled under the disclosure
policy.

## Scope

This repository contains **Agent Skills** — reference documentation and bundled
OpenAPI specs that teach AI coding assistants how to integrate with the
FastBound API. Relevant reports here include, for example, a skill or reference
file that recommends an insecure integration pattern, or a credential-handling
example that leaks a secret.

Vulnerabilities in the **FastBound product or API itself** (cloud.fastbound.com,
the web application, authentication, etc.) are out of scope for this repository
and should go directly through the disclosure policy linked above.
