# 22. Secrets and Auth

How CI authenticates to Salesforce orgs without leaking credentials. Practical patterns. The trade-offs between convenience and security.

## What you need to authenticate

The Salesforce CLI authenticates to an org using one of:

- **Web flow** (browser-based, interactive). Used by humans on local machines.
- **SFDX auth URL** (a string containing a refresh token).
- **JWT bearer flow** (a connected app + cert).
- **Username + password + token** (legacy SOAP login). Avoid.

CI uses the second or third. Web flow is interactive and not suitable for automation.

## SFDX auth URL

The simplest CI auth.

### How it works

After authenticating a user via web flow, the CLI can produce a long-form URL that contains:

- Refresh token.
- Client id.
- Login URL.

The URL is enough to re-establish auth without further interaction.

### Generating

```bash
sf org display --target-org devhub --verbose --json | jq -r '.result.sfdxAuthUrl'
```

Outputs something like:

```
force://PlatformCLI::5Aep861TSESvWeug_zS_5K...@my-dev-hub.my.salesforce.com
```

Save this as a CI secret. Treat it like a password.

### Using in CI

```bash
echo "$SFDX_AUTH_URL" > /tmp/auth.txt
sf org login sfdx-url --sfdx-url-file /tmp/auth.txt --alias my-org --set-default
```

The CLI authenticates using the URL.

### Pros

- One-step auth.
- Works in any CI provider.
- No certificate management.

### Cons

- Refresh tokens can be revoked, expire if unused, or rotate.
- The URL is a credential. Anyone with it can act as that user.
- Hard to audit who used the credential.

For non-production CI (PR validation, integration deploys), SFDX auth URL is fine. For production, JWT is better.

## JWT bearer flow

The recommended production CI auth.

### How it works

A connected app on Salesforce has a public certificate. CI signs a JWT with the matching private key and exchanges it for a short-lived access token.

No refresh tokens. No long-term credentials in CI's hands. Just a certificate that you can rotate.

### Setup, one time

1. Generate a key pair:

```bash
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
openssl rsa -passin pass:x -in server.pass.key -out server.key
openssl req -new -key server.key -out server.csr
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
```

2. Create a connected app in Salesforce:
   - Setup, App Manager, New Connected App.
   - Enable OAuth Settings.
   - Callback URL: anything (not used for JWT).
   - Selected OAuth Scopes: `api`, `refresh_token`, `web`.
   - Use digital signatures: upload `server.crt`.
   - Save.

3. Approve the connected app for the user CI will run as:
   - Setup, Manage Connected Apps.
   - Edit Policies.
   - Permitted Users: Admin approved users are pre-authorized.
   - Save.
   - Add the CI user via Profiles or Permission Sets.

4. Note the Consumer Key (Client ID) from the connected app.

### Using in CI

Store as secrets:

- `SF_CLIENT_ID`: the Consumer Key.
- `SF_USERNAME`: the username CI runs as.
- `SF_JWT_KEY`: the contents of `server.key`.

In CI:

```bash
echo "$SF_JWT_KEY" > /tmp/server.key
sf org login jwt \
    --client-id "$SF_CLIENT_ID" \
    --jwt-key-file /tmp/server.key \
    --username "$SF_USERNAME" \
    --instance-url "$SF_INSTANCE_URL" \
    --alias prod-deploy \
    --set-default
```

The CLI generates an access token. Use it for the deploy. It expires; the next CI run gets a new one.

### Pros

- No long-lived credentials in CI.
- Certificate rotation is straightforward.
- Audit trail: connected app shows usage.
- Best practice for production-targeted CI.

### Cons

- More setup (cert, connected app, profile).
- Cert management responsibility.
- Slightly more debugging when it fails.

## Org isolation in CI

Different environments need different credentials. Plan for separation:

- `SFDX_AUTH_URL_DEVHUB`: Dev Hub for scratch creation.
- `SFDX_AUTH_URL_INTEGRATION`: integration sandbox.
- `SFDX_AUTH_URL_QA`: QA sandbox.
- `JWT_KEY_PRODUCTION`, `JWT_CLIENT_ID_PRODUCTION`, etc.: production via JWT.

Don't share credentials across environments. A compromise of integration shouldn't compromise production.

## Where to store secrets

### GitHub Actions

`Settings, Secrets and variables, Actions, New repository secret`.

For sensitive secrets, use environments with required reviewers. Production-targeted secrets go behind an environment that requires manual approval.

### GitLab CI

`Settings, CI/CD, Variables`. Mark variables as `Masked` and `Protected` (only available on protected branches).

### Bitbucket Pipelines

`Repository settings, Pipelines, Repository variables`. Mark as `Secured`.

### Azure DevOps

`Pipelines, Library, Variable groups`. Mark variables as secret.

### Common rules

- **Never commit secrets to the repo.** No `.env` files. No "I'll remove it later" commits.
- **Rotate periodically.** Quarterly is fine for most. Monthly for high-value targets.
- **Audit access.** Who has access to view or edit secrets? Should be a small list.
- **Separate dev and prod.** Production secrets visible only to deploy approvers.

## Connected app permissions

The connected app's policy decides what the auth grants. Configure tightly:

- **OAuth Scopes**: include only what's needed. `api` is usually enough. `refresh_token` if using refresh-token-based auth.
- **Permitted Users**: prefer "Admin approved users are pre-authorized". Then assign the connected app to a specific profile or permission set, used by CI users only.
- **IP Restrictions**: restrict to your CI provider's IP ranges if possible.

Don't use a generic admin user for CI. Use a dedicated CI user with the minimum permissions.

## CI user identity

A common pattern: a dedicated `ci@yourdomain.com` user in production. The user has:

- A profile with deploy permissions, no UI access.
- An IP restriction to CI provider IPs.
- Permission sets matching what CI deployments need.

CI authenticates as this user. Audit logs show CI's actions distinctly from human deploys.

For production: a "DevOps" user. For sandbox CI: same user, scoped down.

## Rotating credentials

### SFDX auth URL

Re-authenticate the user, regenerate the URL, update the CI secret.

```bash
sf org login web --alias prod
sf org display --target-org prod --verbose --json | jq -r '.result.sfdxAuthUrl'
# Update SFDX_AUTH_URL_PROD in CI
```

### JWT certificate

Generate a new key pair, upload the new cert to the connected app (you can have multiple certs simultaneously during transition), update the CI secret with the new key, then revoke the old cert from the connected app.

### When to rotate

- Quarterly (default).
- Immediately upon any suspicion of compromise.
- When a CI provider has a security incident.
- When a team member with access leaves.

## Local developer auth

Developers don't need CI credentials. They authenticate via web flow:

```bash
sf org login web --set-default-dev-hub --alias devhub
sf org login web --alias my-sandbox
```

Tokens persist on the local machine via the CLI's keyring. They don't need to be in environment variables.

If a developer leaves: revoke their tokens by invalidating their user's connected app session in Salesforce.

## Anti-patterns

### Hardcoded credentials

A `.env` file in the repo with real tokens. Auto-fail your build if you find these.

### One credential for all environments

`SF_AUTH_URL` that works against any org. Compromise of any environment compromises all.

### Auth in plaintext logs

CI scripts that `echo` the credential for debugging. Even in build logs, this leaks.

### Long-running access tokens

Trying to extend access token life beyond Salesforce's defaults. Don't. Refresh as needed.

### Sharing CI accounts across teams

Multiple teams using the same CI user. Audit logs lose meaning.

## A safe CI auth checklist

- [ ] Secrets stored in CI provider's secrets manager, not in the repo.
- [ ] Production uses JWT, not SFDX auth URL.
- [ ] Different secrets per environment.
- [ ] CI user is dedicated, not a personal admin user.
- [ ] CI user's profile is scoped to deployment-only permissions.
- [ ] Connected app for JWT has IP restrictions where possible.
- [ ] Production deploys go behind required-approver gates.
- [ ] Credentials rotate at least quarterly.
- [ ] No credentials in code, logs, or chat messages.
- [ ] Documented runbook for credential rotation.

## References

- [Salesforce CLI authentication](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth.htm)
- [JWT bearer flow](https://help.salesforce.com/s/articleView?id=sf.remoteaccess_oauth_jwt_flow.htm)
- [Connected apps](https://help.salesforce.com/s/articleView?id=sf.connected_app_overview.htm)
- [GitHub Actions secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [GitLab CI variables](https://docs.gitlab.com/ee/ci/variables/)
