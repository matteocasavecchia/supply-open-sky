# Security Policy

Supply Open Sky is a system designed for safety-critical operations in
environments with limited margin for error. We take security and safety
concerns seriously, both for the project itself and for the people the
system is intended to serve.

This document describes how to report security-relevant issues, what is in
scope for this channel, and what to expect after a report is submitted.

---

## Scope

This security policy covers:

- **Documentation security.** Information published in this repository that
  should not have been (e.g. unintended exposure of personal data,
  operational details, or location-specific information).
- **Future code releases.** Vulnerabilities in software components published
  by the project, once such components are released publicly.
- **Project infrastructure.** Compromise or misuse of project communication
  channels, repository access, contact addresses, and similar.
- **Substantive design concerns.** Vulnerabilities in the architecture or
  protocols described in the public documentation, when reported by
  researchers or engineers in good faith with technical substance.

This security policy **does not cover**:

- Operational deployment-specific concerns (specific people, specific
  locations, specific incidents in the field). Those require channels
  appropriate to the deployment context and are not handled here.
- General feedback on system design unrelated to security or safety.
  Please use the regular [issue tracker](https://github.com/matteocasavecchia/supply-open-sky/issues)
  for that.
- Vulnerabilities in third-party software the project may reference but
  does not maintain.

## How to report

Two channels are available; choose whichever you prefer.

**Option 1 — GitHub Private Vulnerability Reporting.**
Use GitHub's built-in private reporting at
[github.com/matteocasavecchia/supply-open-sky/security](https://github.com/matteocasavecchia/supply-open-sky/security).
This creates a private discussion accessible only to the project
maintainers and to you. Recommended for reports involving code or
repository content.

**Option 2 — Email.**
Send a report to
[matteo.casavecchia+sos-security@gmail.com](mailto:matteo.casavecchia+sos-security@gmail.com).
Recommended if you do not have a GitHub account, or for reports that do
not fit the GitHub flow.

Please **do not open a public issue** for security-relevant reports.

### What to include

A useful report typically contains:

- A clear description of the issue and why it is security-relevant.
- The location of the affected content (file, section, URL, or component).
- Steps or context needed to understand or verify the issue.
- Any suggested mitigation, if applicable.
- Whether you wish to be credited publicly once the issue is addressed
  (and how — name, handle, affiliation).

You do not need to provide a fix. A clear description of the issue is
enough.

## What to expect

We aim to:

- **Acknowledge** receipt of a security report within **7 days**.
- Provide an **initial assessment** of the report within **14 days**, including
  whether we consider it in scope and a preliminary view on severity.
- Communicate progress on remediation as it advances.
- Coordinate disclosure with the reporter when the issue is resolved.

These timelines reflect the current capacity of the project. We will be
transparent if a particular report requires more time, and we will
communicate why.

For issues we determine to be out of scope or not security-relevant, we
will say so explicitly and, where possible, redirect to a more appropriate
channel.

## Coordinated disclosure

We follow a **coordinated disclosure** approach. If you have reported an
issue:

- Please give the project a reasonable time to investigate and respond
  before discussing the issue publicly.
- We will work with you to agree on a disclosure timeline appropriate to
  the severity of the issue and the complexity of remediation.
- For documentation-only issues that pose immediate risk (e.g. accidentally
  published personal data), we will act quickly and coordinate disclosure
  on a compressed timeline.

We do not currently operate a bug bounty program. Public credit is given
on request, subject to the reporter's preference.

## Out-of-band contact

If for any reason both reporting channels above are unavailable or
compromised, you can reach the maintainer through the contact information
on
[Matteo Casavecchia's LinkedIn profile](https://www.linkedin.com/in/casavecchia/).
This is a fallback only; please use the primary channels above whenever
possible.
