# Security policy

## Reporting a vulnerability

Please report security problems privately through GitHub's private vulnerability reporting: open the
**Security** tab of this repository and choose **Report a vulnerability**
(`https://github.com/OWNER/REPO/security/advisories/new`).

Please do not open a public issue or post details publicly until a fix is out. You can expect an
acknowledgement within a week. This is a small project maintained in spare time, so fixes are best effort.

## Supported versions

Only the latest release (and the default branch) receives security fixes.

## What is checked automatically

Every change runs static analysis (Semgrep, CodeQL), dependency and secret scanning, and a weekly OpenSSF
Scorecard. Released container images are scanned, signed with Sigstore cosign, and carry an SBOM and build
provenance attestation.
