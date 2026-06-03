# Migrating Secrets to SOPS

In the past I used `.env` files with `KEY=VALUE` pairs to store secrets locally (outside the `git`
repo). In CI I used the same or similar key-value pairs.

This has some drawback:

- I need to keep the secrets in sync.
- I don't have a history of the changes: I can't see at what time a secret got update. This can lead
  to uncertainty, when you work in a team. Example: Today a secret does no longer work. Did someone
  update the secret? Nobody knows...
- I can't easily look at the secrets in CI. Platforms like Github/Forgejo do not show you the value
  in the UI.

The solution is: [SOPS](https://getsops.io/) (*S*ecrets *OP*eration*S*)

> SOPS encrypts configuration files while keeping the structure visible. Keys are not encrypted,
> while values and comments are encrypted. This allows you to understand the configuration without
> seeing sensitive values. Also commented-out secrets aren’t suddenly visible to everyone!

Together with [Age](https://github.com/FiloSottile/age) (A simple, modern and secure encryption
tool), you can store the secrets in `git`. The keys are readable, and the values are encrypted with
a `SOPS_AGE_KEY`.

In `.env` and in CI config you only store `SOPS_AGE_KEY`.

## Example

The keys are readable, the values are encrypted:

`secrets.enc.yaml`

```yaml
FOO_API_KEY: ENC[AES256_GCM,data:Dnwp+wSxKWCrWXrOAr0NqD5odZnitL7dUFZBpTmx/vIBv7l/63DU6HDiWgWConkYfGo=,iv:by+yyCzv/jLAm2BQZJIwe9cArms+G2AxmgzGRketCfQ=,tag:1Wj/Em39+3FeBqUjkQouDQ==,type:str]
WIREGUARD_PRIVATE_KEY_BAR: ENC[AES256_GCM,data:u3GNdUsUWcwkRxjrfQAkUty0P3m4axoTTmK8Hhnfy5dV7r3s/IP4mWqS25o=,iv:mHFQODMqJD/VVM0udpyyz3qEt4EZCSquqqurwhC/Hsw=,tag:L/abimAshyCm4wyG1h2Jag==,type:str]
sops:
    age:
        - enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSA1c1o3dzRzYndUVUplSTVB
            MjFsZ0Z4MmpBaXZxTys5SEFKa2VjeUJNVVZZCjI2b3MrSWg5MEtVN3ZLZ2FDZHNu
            OTM0QXBlUlRJcEdYM2hvWnhGL2JxUVkKLS0tIFB4a1dQNGtoRnFXdUVRSmpneDl3
            NVF4N1dlaEtMQmZZSlFmamRMWUdsem8K38dzAcQNcZnOZztJQ/fHlXTbkG09GF71
            V0njc2VB7Way3NuYjgXdHhYESiX92W6NMUaK0zzED5Q7jVm4D14AHg==
            -----END AGE ENCRYPTED FILE-----
          recipient: age1r0k34dkgzppaew7etm3ka7p0dgxcd365gxe66kuuqsnw6hqax9qswda0sh
    lastmodified: "2026-06-02T11:12:37Z"
    mac: ENC[AES256_GCM,data:KJKlLtQqxzdwW68GEblyknEKE7xNpEc8YGpNiZzNY+6c4RLOknkP2Bm7eEQei47iGadVFJGu9o33aPgas+DCu5kZgJkl13dRT3y5/JxgfmQeeOPLsV+sDMfErLPeaqT6OX4JSWkBbL5Hu7RheY2hNAFtqHkqceo0iMKq91U4xwU=,iv:rGK8n4eutmgOe/5cZeWEU9heb/mU0v5eArq/wyTFjaI=,tag:8dRbB4KgSg0I5aCLJVl7EA==,type:str]
    unencrypted_suffix: _unencrypted
    version: 3.13.1
```

You can edit the values easily:

Install `sops` cli. For example via `nix profile add nixpkgs#sops`.

```bash
sops secrets.enc.yaml
```

[Docs about `sops` and env var SOPS_AGE_KEY](https://getsops.io/docs/#encrypting-using-age)

## Switching from Secrets in CI to SOPS+Age

- [`age`](https://github.com/FiloSottile/age) installed locally
- [`sops`](https://github.com/getsops/sops) installed locally
- Access to add/remove secrets in your CI provider
- A CI pipeline you can trigger manually

## Step 1 — Generate an AGE key pair

```bash
age-keygen -o age.key
```

This writes a file like:

```yaml
# created: 2026-06-02T10:00:00Z
# public key: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AGE-SECRET-KEY-1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Keep this file safe. The **public key** goes into `.sops.yaml`. The **private key** (the
`AGE-SECRET-KEY-1...` line) becomes the CI secret `SOPS_AGE_KEY`.

---

## Step 2 — Add SOPS_AGE_KEY to CI

In your CI provider's secret store, create one secret:

| Name | Value |
|------|-------|
| `SOPS_AGE_KEY` | The full `AGE-SECRET-KEY-1...` line from `age.key` |

This is the **only** CI secret you will need going forward.

---

## Step 3 — Configure SOPS to use that key

Create `.sops.yaml` at the root of your repository:

```yaml
creation_rules:
  - path_regex: secrets\.yaml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Replace the `age1...` value with the public key from `age.key`.

Commit `.sops.yaml` to the repository — it contains only the public key and is safe to commit.

---

## Step 4 — Recover the current secret values from CI

Because CI secret values are write-only (not readable via UI or API), use the CI runner itself to
reveal them safely.

Write a temporary pipeline job that encrypts and prints every secret you want to migrate. Replace
`age1xxxxxxxx...` with your actual public key. The exact syntax depends on your CI provider:

**GitHub Actions (`.github/workflows/dump-secrets.yml`):**

```yaml
name: Dump secrets (temporary – delete after use)
on:
  workflow_dispatch:

jobs:
  dump:
    runs-on: ubuntu-latest
    steps:
      - name: Install age
        run: sudo apt-get update && sudo apt-get install -y age
      - name: Print secrets safely
        env:
          SECRET_FOO: ${{ secrets.FOO }}
          SECRET_BAR: ${{ secrets.BAR }}
          # … add every secret you want to migrate
        run: |
          {
            echo "FOO=$SECRET_FOO"
            echo "BAR=$SECRET_BAR"
          } | age -r age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx -a
```

Trigger this job once (manually via the UI) and copy the encrypted ASCII-armored output. Then
**immediately delete this pipeline file** — do not leave it in the repository.

> Note: Save the output locally to a file (e.g., `secrets.txt.age`) and decrypt it using your
> private key: `echo $SOPS_AGE_KEY | age -d -i - secrets.txt.age > secrets.txt` This ensures your
> secrets are never exposed in plaintext in the CI logs!

---

## Step 5 — Create and encrypt secrets.yaml

Using the values collected in Step 4, create a plaintext `secrets.yaml` locally:

```yaml
FOO: "value-of-foo"
BAR: "value-of-bar"
DATABASE_URL: "postgres://..."
# … all remaining secrets
```

Encrypt it with SOPS:

```bash
sops --encrypt --age age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  secrets.yaml > secrets.enc.yaml
```

Or, if you prefer to edit in place:

```bash
sops secrets.yaml   # opens editor; save to encrypt on exit
```

Verify the encrypted file looks correct (should be YAML with `sops:` metadata block at the bottom).

When `SOPS_AGE_KEY` is set, you can edit the unencrypted file like this:

```bash
sops secrets.enc.yaml
```

---

## Step 6 — Commit the encrypted file

```bash
git add .sops.yaml secrets.enc.yaml
git commit -m "chore: add SOPS-encrypted secrets"
git push
```

The encrypted file is safe to store in a public or private repository.

---

## Step 7 — Update your CI pipeline to use SOPS

In every job that needs secrets, decrypt `secrets.enc.yaml` and source the values. `SOPS_AGE_KEY` is
already set as a CI secret from Step 2.

**Shell snippet (all providers):**

```bash
# Install sops if not in the runner image
# Replace <tag> with the latest version from https://github.com/getsops/sops/releases
SOPS_TAG=$(curl -s https://api.github.com/repos/getsops/sops/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
curl -Lo /usr/local/bin/sops \
  "https://github.com/getsops/sops/releases/download/${SOPS_TAG}/sops-${SOPS_TAG}.linux.amd64"
chmod +x /usr/local/bin/sops

# Decrypt and export all keys as environment variables
export $(sops --decrypt secrets.enc.yaml | \
  grep -v '^#' | xargs)
```

Or parse with `yq` for more control:

```bash
eval "$(sops --decrypt secrets.enc.yaml | \
  yq e 'to_entries | .[] | "export " + .key + "=" + (.value | @sh)' -)"
```

---

## Step 8 — Remove all old CI secrets

Once the pipeline is running correctly with SOPS-decrypted values, delete every CI secret **except
`SOPS_AGE_KEY`** from the provider's secret store.

| Provider | Where to delete |
|----------|----------------|
| GitHub | Settings → Secrets and variables → Actions |
| Codeberg/Gitea | Settings → Secrets |
| GitLab | Settings → CI/CD → Variables |

---

## Security notes

- Never commit the plaintext `secrets.yaml` or `age.key`.
- The `SOPS_AGE_KEY` CI secret is the single point of trust. Rotate it by generating a new key pair,
  updating `.sops.yaml`, running `sops updatekeys`, and replacing the CI secret.
- Keep a backup of `age.key` in a password manager or secrets vault outside the repository.

Additionally, it is recommended to have a [pre-commit](https://pre-commit.com) configuration, which
checks for secrets before you accidentally commit them. Example:

```yaml
# See https://pre-commit.com/hooks.html for more hooks
repos:
- repo: https://github.com/pre-commit/pre-commit-hooks
  ...

- repo: https://github.com/gitleaks/gitleaks
  hooks:
  - id: gitleaks
```
