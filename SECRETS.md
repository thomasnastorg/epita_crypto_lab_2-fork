# SECRETS.md: how secrets are managed in this repository

## Recipients

| Identity | Public key | prod | dev |
|---|---|---|---|
| alice | `age19eft4yajxnlu4n7q9j25pftt73lgcmtmq4zfyxnzf2aldtgtrvzsjscfwh` | yes | yes |
| bob | `age1np6jd0w8r955tc07mc84y07zuvzfqwq3px8lrt79vp8eq76e7e3q74fddc` | no | yes |
| ci | `age18h5lakt08jldmmr4rgtarggf74945r8e03q0cg0kuw5ql88f9e9qq6uz9r` | yes | no |
| carol | `age1ryd237p59dnnes3vhxujtsh8ww480pcsunr6mzyq0uygsugveezqmcndjy` | **removed** | **removed** |

Rules live in `.sops.yaml` (one rule per path: `secrets/prod/`, `secrets/dev/`).
Private keys (`*.key`) are git-ignored and never committed. The CI private key is stored only
in the `SOPS_AGE_KEY` repository secret.

Developer setup, once per clone: `git config core.hooksPath .githooks` (betterleaks pre-commit hook).

## Step 1 · age and recipients

`step1/secret.age` is `"db password: hunter2"` encrypted to alice, bob and carol.

- **How many times is the payload encrypted?** Once. A random 128-bit *file key* is generated,
  and the payload is encrypted a single time with a key derived from it (ChaCha20-Poly1305, STREAM, 64 KiB chunks).
- **What is duplicated per recipient?** Only the file key: it is wrapped once per recipient, giving
  one `-> X25519 ...` stanza per recipient in the header (3 stanzas here).
- **What is in each recipient block?** `-> X25519 <ephemeral public key>` followed by the wrapped
  file key. For each recipient, age makes a fresh ephemeral X25519 key pair, does ECDH with the
  recipient's public key, derives a wrapping key with HKDF-SHA256 (salt = ephemeral pub || recipient pub),
  and encrypts the file key with ChaCha20-Poly1305. The header ends with `--- <HMAC>`, a MAC over the
  header keyed from the file key, then the binary payload (nonce + encrypted chunks).
  This is the hybrid "KEM + DEM" construction built by hand in Lab 1.

## Step 2 · Commit a secret, then deal with it

- **The file is deleted in HEAD. If it had been pushed, who can still read the password?**
  Everyone who can read the repository: `git show <commit>:database.yaml` still returns it. That
  includes every clone and fork made since, CI caches, GitHub (cached views, PR refs, the commit URL
  stays reachable by SHA), and any secret scanner/bot that indexed the push. `betterleaks git .`
  found it because it scans every commit, not only HEAD.
- **After `filter-repo`, is the password safe? What had to be done first?** No. `filter-repo` only
  rewrites *our* copy (and the remote after a force-push). Existing clones, forks and caches still
  have the old commits. The password must be considered compromised: **rotate it first** (change it
  on the database), then clean history. The clean-up is hygiene, not remediation.
  In this repo the leaking commits were removed with
  `git filter-repo --path database.yaml --invert-paths --force` before the first push.
- **What does `git commit --no-verify` do to the hook?** It skips it completely: client-side hooks
  are a convenience, not a control (they are also not enabled until each developer runs
  `git config core.hooksPath .githooks`). The same check must run server-side, where the developer
  cannot bypass it: in CI on every push/PR (the `betterleaks dir` step of the workflow) and ideally
  with GitHub push protection / secret scanning.

## Step 3 · SOPS with path rules

- **Decrypt prod with `bob.key`:** refused. `Failed to get the data key required to decrypt the SOPS
  file ... no master key was able to decrypt the file`. Bob has no stanza in the prod file's `sops.age`
  list, so he cannot unwrap the data key. (`./deploy.sh` with bob's key fails on prod and succeeds on dev.)
- **Edit an encrypted value by hand:** `Could not decrypt with AES_GCM: cipher: message authentication
  failed`. Each value is AES-256-GCM with the path of the key as additional data, so it cannot be
  altered or moved. Editing a *cleartext* field (e.g. `port` in dev) gives `MAC mismatch`: the `mac`
  covers the whole document, cleartext values included.
- Keys and comments stay readable, so diffs stay reviewable. In dev, `encrypted_regex` keeps
  `host`/`port`/`user` in clear and encrypts only `password|token|key|secret`.

## Step 4 · Carol leaves (four commits)

1. `carol leaves 1/4`: remove carol from `.sops.yaml`.
2. `carol leaves 2/4`: `sops updatekeys` on every file. Carol's stanza is removed, **but the data key is
   unchanged**, and carol may have already seen it.
3. `carol leaves 3/4`: `sops -r -i`: new data key, every value re-encrypted.
4. `carol leaves 4/4`: **rotate the actual passwords** (prod and dev).

Without step 4, carol keeps access: her clone holds the old commits and `carol.key` still decrypts
them (checked: carol decrypts the pre-rotation prod file from history). Steps 1–3 only stop her
from reading *future* versions. Removing a block never revokes a past read: leaving is a rotation event.

## Step 5 · Decrypt in CI

`ci.key` is a recipient of the prod rule only. Its private key is in the `SOPS_AGE_KEY` Actions secret.
The workflow scans the files with `betterleaks dir`, then `deploy.sh` decrypts and calls the endpoint.

**Break it on purpose:** `echo $SOPS_AGE_KEY` is shown as `***`, but `sops -d secrets/prod/database.yaml`
prints the decrypted password in clear: it is not a registered secret, so GitHub does not mask it.
GitHub replaces only the exact secret string by `***`, and masking is best-effort: `echo "$SOPS_AGE_KEY" | base64`, splitting the value
across lines, `rev`, or a value showing up inside an error message all go through in clear. Anyone
who can push to a branch that triggers the workflow can exfiltrate the key.

- **Where does the CI key live?** In GitHub's secret store, injected into the runner's environment
  for each job. It decrypts production: it is the highest-value credential of the repo.
- **Who can see the runner?** GitHub (hosted runners), anyone with write access (they can change the
  workflow or `deploy.sh`), and anyone who can read the logs.
- **Can a PR from a fork reach it?** Not with this workflow: there is no `pull_request` trigger, and
  GitHub does not pass secrets to `pull_request` runs from forks. `pull_request_target` would be the
  dangerous one. A same-repo branch push to `main` does reach it.
- **Better:** a CI identity in a cloud KMS (AWS/GCP/Azure KMS via OIDC federation), IAM-scoped to this
  repo/branch, no long-lived secret, and an audit trail of every decrypt.
