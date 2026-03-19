# sapl-gitops-demo

End-to-end demonstration of a GitOps pipeline for SAPL authorization policies: write policies, test them in CI, sign and publish bundles, and consume them from a running SAPL Node.

## What this demonstrates

```
git push -> CI tests policies -> signs bundle -> publishes to GitHub Release
                |                                        |
                v                                        v
        coverage report                      SAPL Node downloads
        deployed to Pages                    signed bundle
```

1. **Policy-as-code** -- `.sapl` policies and `.sapltest` tests live in version control
2. **Automated testing** -- CI runs `sapl test` with coverage quality gates on every push
3. **Coverage reporting** -- HTML coverage reports deployed to GitHub Pages
4. **Cryptographic signing** -- Bundles are signed with Ed25519 keys, verified before publish
5. **Artifact distribution** -- Signed bundles published as GitHub Release assets
6. **Secure consumption** -- SAPL Node verifies bundle signatures before loading

## Included policies

| Policy Set | Description |
|-----------|-------------|
| `classified_documents` | NATO clearance-level document filtering based on time windows |
| `patients` | Medical record transformation with blackening, deletion, and replacement |
| `demo_set` | Time-based permit/deny transitions with logging obligations, email notifications, and resource transformation |
| `argumentmodification` | Method argument modification via obligations |

All policies include streaming test scenarios demonstrating SAPL's reactive decision updates.

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

Enable GitHub Pages:

- Go to Settings > Pages
- Set Source to "GitHub Actions"

The public key `signing.pub` is committed to the repository for verification.

### 3. Push a policy change

Edit any file under `policies/` and push to `main`. The pipeline will:

1. Run all `.sapltest` files against the policies
2. Enforce coverage quality gates
3. Deploy the HTML coverage report to GitHub Pages
4. Create and sign a `.saplbundle`
5. Verify the signature with the committed public key
6. Publish the bundle to the `latest-bundle` GitHub Release

### 4. View coverage report

After a successful push to `main`, the coverage report is available at:

**[Coverage Report](https://heutelbeck.github.io/sapl-gitops-demo/)**

### 5. Consume the bundle

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
  classified_documents.sapl       NATO clearance-level filtering
  classified_documents.sapltest   Clearance tests with streaming
  patients.sapl                   Medical record transformation
  patients.sapltest               Transformation tests with streaming
  demo_set.sapl                   Time-based permit/deny with obligations
  demo_set.sapltest               Time window and scope tests
  argumentmodification.sapl       Argument modification via obligations
  argumentmodification.sapltest   Obligation structure tests
  pdp.json                        PDP configuration
signing.pub                       Ed25519 public key (for verification)
.github/workflows/
  policy-pipeline.yml             CI/CD pipeline
```

## Pull request workflow

On pull requests, the pipeline runs tests only (no bundle publish, no Pages deploy). Reviewers see test results and coverage artifacts before merging policy changes.

## License

Apache License 2.0
