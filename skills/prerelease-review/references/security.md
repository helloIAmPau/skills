# White-box and black-box assessment

## Scope and evidence

Derive assets, roles, trust boundaries and exposed interfaces from the project.
Record the tested candidate SHA, environment and target URLs. Use a local,
disposable environment with fictional accounts and data by default. If the
candidate cannot be run, complete source review and mark dynamic checks blocked.

Inspect code/configuration in white-box review. Independently probe observable
responses and side effects through public interfaces in black-box review;
source review and in-process helper tests do not substitute for this. Map each
applicable boundary to an expected security property and the observed result.

Run bounded probes only against the project's authorized test targets. Live or
third-party targets, disruptive load and destructive techniques need separate
scope authorization. Keep email, payments and other integrations on test sinks.
Restore review-created state and stop review-started services afterward.

## White-box work

- Review authentication and recovery, session/token handling, authorization at
  object/role/tenant boundaries, and sensitive fields in responses, logs or errors.
- Trace untrusted inputs to database, shell, template, HTML, file and network
  sinks. Assess applicable injection, XSS, CSRF/CORS, SSRF, traversal/upload,
  deserialization and redirect risks against actual data flow and controls.
- Review validation, resource limits, replay/race behavior, cryptography and
  secret handling, including tracked history and build artifacts. Report secret
  locations/types and remediation without reproducing secret values.
- Inspect deployment and CI configuration, exposed services, debug settings,
  service privileges, image/runtime versions, dependency provenance and lockfiles.
  Distinguish test-only settings from settings shipped to production.
- Use available ecosystem vulnerability and static-analysis tools without auto-fix
  or dependency upgrades. Assess direct/transitive production and development
  dependencies, including build/CI exposure. Record scanner/database timestamps,
  coverage and failures. Check CVE/advisory applicability and fixed versions
  against current primary vendor/advisory sources; do not invent advisory IDs
  or rely on remembered vulnerability status. Explain reachability and uncertainty.

## Black-box work

Select probes from the actual attack surface. For a web service, normally cover
unauthenticated access, account A accessing account B's objects, privilege
boundaries, session expiry/logout/replay, invalid inputs and sensitive response
fields. Where relevant, also check injection payloads, origin/CSRF behavior,
error leakage, published files, redirects, uploads and bounded rate/size limits.

Use at least two fictional identities when testing object ownership and the
relevant roles when testing privilege boundaries. Authenticate through supported
public flows. A refusal status alone is not enough: verify that the operation
did not read or change the protected data.

Record each case's identity/role, preconditions, endpoint/action, sanitized
request, expected result, actual response and relevant state observation.
A scanner finding needs manual triage; scanner absence or a tool failure is a
coverage limitation. Record black-box cases not executable in the environment.

Test the deployed configuration only if that environment is in scope. A local
HTTP stack cannot establish production TLS, reverse-proxy or cloud security;
mark those properties unverified rather than as failed or passed by assumption.

## Findings and disclosure

Use stable IDs, impact-based severity (critical/high/medium/low/informational)
and separate confidence (confirmed/suspected). Include attack prerequisites,
affected versions/components, reproduction evidence, likely impact, remediation
and the check that would verify a fix. Distinguish new regressions from existing
issues and product risks from development-only exposure.

Keep credentials, personal data and usable sensitive exploit details out of
public issues and logs. Publish a redacted summary and use an existing authorized
private reporting channel for restricted evidence. If none exists, retain the
safe public report and request a private destination for the restricted details;
do not publish a security advisory or notify others without authorization.

[OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
provides applicable web-testing categories; use the project's actual surface
rather than claiming every category was tested.
