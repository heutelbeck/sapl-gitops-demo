# sapl-gitops-demo

End-to-end demonstration of a GitOps pipeline for SAPL authorization policies: write policies, test them in CI, sign and publish bundles, and consume them from a running SAPL Node.

## What this demonstrates

```
git push -> CI tests policies -> signs bundle -> publishes to GitHub Release
                                                        |
                                          SAPL Node downloads signed bundle
```

1. **Policy-as-code** -- `.sapl` policies and `.sapltest` tests live in version control
2. **Automated testing** -- CI runs `sapl test` with coverage quality gates on every push
3. **Cryptographic signing** -- Bundles are signed with Ed25519 keys, verified before publish
4. **Artifact distribution** -- Signed bundles published as GitHub Release assets
5. **Secure consumption** -- SAPL Node verifies bundle signatures before loading

## Setup

### 1. Generate signing keypair

```
sapl bundle keygen -o signing
```

This creates `signing.pem` (private key) and `signing.pub` (public key).

### 2. Configure GitHub

Add the private key as a repository secret:

- Go to Settings > Secrets and variables > Actions
- Create secret `BUNDLE_SIGNING_KEY` with the contents of `signing.pem`

The public key `signing.pub` is committed to the repository for verification.

### 3. Push a policy change

Edit any file under `policies/` and push to `main`. The pipeline will:

1. Run all `.sapltest` files against the policies
2. Enforce coverage quality gates
3. Create and sign a `.saplbundle`
4. Verify the signature with the committed public key
5. Publish the bundle to the `latest-bundle` GitHub Release

### 4. Consume the bundle

Download the latest signed bundle from the release:

```
gh release download latest-bundle --repo <owner>/sapl-gitops-demo --pattern "*.saplbundle"
```

Run a SAPL Node with the bundle:

```
sapl --bundle policies.saplbundle
```

Or verify it independently:

```
sapl bundle verify -b policies.saplbundle -k signing.pub
sapl bundle inspect -b policies.saplbundle
```

## Repository structure

```
policies/
  api-access.sapl          Policy documents
  api-access.sapltest      Test scenarios
  pdp.json                 PDP configuration (combining algorithm)
signing.pub                Ed25519 public key (for bundle verification)
.github/workflows/
  policy-pipeline.yml      CI/CD pipeline
```

## Pull request workflow

On pull requests, the pipeline runs tests only (no bundle publish). Reviewers see test results and coverage reports before merging policy changes.

## License

Apache License 2.0
