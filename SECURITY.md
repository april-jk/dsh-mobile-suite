# Security Policy

## Supported versions

Only the latest GitHub Release receives security fixes during the MVP phase.

## Reporting a vulnerability

Use [GitHub private vulnerability reporting](https://github.com/april-jk/dsh-mobile-suite/security/advisories/new) for vulnerabilities affecting the coordinated product. Do not open a public issue with credentials, personal data, exploit details, or a working proof of concept.

Include the affected component and version, deployment model, reproduction steps, expected impact, and any suggested mitigation. Reports are acknowledged on a best-effort community-maintained basis.

For a vulnerability isolated to one component, the corresponding component repository may also be used if it has private reporting enabled.

## MVP limitations

Current Mobile and Browser clients use the `sealed-tunnel-v1` application-layer E2EE profile. The Relay routes ciphertext and can still observe account/device associations, online state, connection times, ciphertext length, and traffic timing.

The Relay-hosted Browser JavaScript is part of the trusted endpoint. Browser E2EE protects against Relay data-plane inspection and accidental persistence, but not a malicious Relay deployment that replaces client code. Use a separately signed native client or browser extension when the web host itself is outside the trust boundary.

## Credential handling

- Store production `JWT_SECRET`, `ADMIN_PASSWORD`, Android signing passwords, npm tokens, and Railway credentials only in the deployment platform or GitHub Actions secret manager.
- Never commit `.env*`, `android/key.properties`, signing keys, provisioning profiles, Firebase service files, SQLite databases, or Companion configuration. The tracked `.env.example` and `key.properties.example` files must contain placeholders only.
- Keep Companion credentials at the default `~/.dsh-remote/config.json` path. Do not point `DSH_REMOTE_CONFIG` into a Git worktree.
- Before pushing a release, run Gitleaks against the root and every component repository:

```bash
for repo in . dsh-mobile dsh-plugin dsh-relay dsh-website; do
  gitleaks git --redact "$repo"
done
```

The published E2EE known-answer vector is deterministic test data and may be reported by generic secret heuristics. Review findings by path and rule; do not globally disable the matching rule.

If a real credential is committed, revoke or rotate it immediately before rewriting Git history. Deleting it in a later commit does not remove it from earlier commits, forks, caches, or CI logs.
