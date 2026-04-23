# sf-jwt-action

A GitHub composite action that installs the Salesforce CLI and authenticates to a Salesforce org using JWT bearer flow.

## Prerequisites

### 1. Generate an RSA key pair

Run the following commands locally to generate a private key and a self-signed certificate:

```bash
openssl genrsa -out server.key 2048
openssl req -new -x509 -key server.key -out server.crt -days 365
```

- `server.key` — your private key. Keep this secret and never commit it.
- `server.crt` — the certificate you will upload to Salesforce.

### 2. Create a Connected App in Salesforce

1. In Salesforce, go to **Setup > App Manager > New Connected App**.
2. Enable **OAuth Settings**.
3. Check **Enable Digital Signatures** and upload `server.crt`.
4. Add the OAuth scope **Manage user data via APIs (api)** (add others as needed).
5. Save the app. Copy the **Consumer Key** — you will need it as a secret.
6. Go to **Manage > Edit Policies** and set **IP Relaxation** to *Relax IP restrictions* (required for JWT flow from GitHub-hosted runners).

### 3. Pre-authorize the user

The Salesforce user must be pre-approved to use the Connected App without interactive login:

- Go to **Setup > Connected Apps > Manage** for your app.
- Under **Permitted Users**, select *Admin approved users are pre-authorized*.
- Assign the profile or permission set of the user you will authenticate with.

### 4. Add secrets to your GitHub repository

Go to **Settings > Secrets and variables > Actions** and add:

| Secret name | Value |
|---|---|
| `SF_CONSUMER_KEY` | Consumer Key from the Connected App |
| `SF_JWT_PRIVATE_KEY` | Full contents of `server.key` (PEM text, including header/footer lines) |
| `SF_USERNAME` | Salesforce username (e.g. `deploy@mycompany.com`) |
| `SF_INSTANCE_URL` | Your org URL (e.g. `https://mycompany.my.salesforce.com`) |

> Store the instance URL as a secret so it is not exposed in workflow logs or version history.

---

## Usage

```yaml
- uses: your-org/sf-jwt-action/.github/actions/sf-auth@main
  with:
    consumer-key: ${{ secrets.SF_CONSUMER_KEY }}
    jwt-private-key: ${{ secrets.SF_JWT_PRIVATE_KEY }}
    username: ${{ secrets.SF_USERNAME }}
    instance-url: ${{ secrets.SF_INSTANCE_URL }}
```

After this step runs, the SF CLI is on `PATH` and the org is set as the default (alias `myorg`), so subsequent steps can run `sf` commands without re-authenticating.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `consumer-key` | Yes | — | Connected App consumer key (client ID) |
| `jwt-private-key` | Yes | — | RSA private key contents (PEM format) |
| `username` | Yes | — | Salesforce username to authenticate as |
| `instance-url` | Yes | — | Salesforce org URL — use a secret (must start with `https://`) |
| `sf-cli-version` | No | `latest` | Salesforce CLI version to install (e.g. `2.50.6`) |

---

## Example workflow

```yaml
name: Deploy to Salesforce

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/sf-jwt-action/.github/actions/sf-auth@main
        with:
          consumer-key: ${{ secrets.SF_CONSUMER_KEY }}
          jwt-private-key: ${{ secrets.SF_JWT_PRIVATE_KEY }}
          username: ${{ secrets.SF_USERNAME }}
          instance-url: ${{ secrets.SF_INSTANCE_URL }}

      - name: Deploy metadata
        run: sf project deploy start --source-dir force-app
```

---

## Security notes

- The private key is written to `$RUNNER_TEMP/server.key` with `chmod 600` and deleted after authentication, even if the step fails (`if: always()`).
- Store the private key, consumer key, username, **and instance URL** as GitHub secrets — never hard-code them in workflow files.
- The `instance-url` is validated at runtime to start with `https://` to prevent accidental plaintext connections.

## CLI caching

The action caches the Salesforce CLI under `~/.npm-global` keyed by OS and the requested version, so repeated runs on the same runner skip the npm install step.
