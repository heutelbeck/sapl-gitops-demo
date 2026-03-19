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

## Quick start

### 1. Fork or clone the repository

Fork this repository on GitHub, or clone and push to your own repo:

```
git clone https://github.com/heutelbeck/sapl-gitops-demo.git
cd sapl-gitops-demo
```

### 2. Install the SAPL CLI

Download the latest binary from [SAPL releases](https://github.com/heutelbeck/sapl-policy-engine/releases/tag/snapshot-node) for your platform and add it to your PATH.

Verify:

```
sapl --version
```

### 3. Generate a signing keypair

```
sapl bundle keygen -o signing
```

This creates `signing.pem` (private key) and `signing.pub` (public key).

### 4. Configure your GitHub repository

**Add the signing secret:**

1. Go to Settings > Secrets and variables > Actions
2. Create a new repository secret named `BUNDLE_SIGNING_KEY`
3. Paste the contents of `signing.pem` as the value

**Enable GitHub Pages:**

1. Go to Settings > Pages
2. Set Source to **GitHub Actions**

**Commit the public key** (if you generated a new keypair):

```
git add signing.pub
git commit -m "add signing public key"
git push
```

### 5. Run the pipeline

Push any change to `policies/` on `main`, or trigger manually:

1. Go to Actions > Policy Pipeline
2. Click "Run workflow"

The pipeline will:

1. Run all `.sapltest` files against the policies
2. Enforce coverage quality gates (100% policy hit ratio)
3. Deploy the HTML coverage report to GitHub Pages
4. Create and sign a `.saplbundle`
5. Verify the signature with the committed public key
6. Publish the bundle to the `latest-bundle` GitHub Release

### 6. View the coverage report

After a successful run, the coverage report is available at:

```
https://<your-username>.github.io/sapl-gitops-demo/
```

### 7. Run a local SAPL Node with the published bundle

Download the signed bundle from your fork's release:

```
gh release download latest-bundle --pattern "*.saplbundle"
gh release download latest-bundle --pattern "signing.pub"
```

Verify the bundle signature:

```
sapl bundle verify -b policies.saplbundle -k signing.pub
```

Inspect its contents:

```
sapl bundle inspect -b policies.saplbundle
```

Start a SAPL Node serving the bundle:

```
sapl --bundle policies.saplbundle --no-verify
```

The PDP server starts on `https://localhost:8443`. Test it with:

```
sapl decide-once --remote -s '{"role":"admin"}' -a '"read"' -r '"document"'
```

### 8. Make a policy change

Edit a policy, push to `main`, and watch the pipeline run. Download the new bundle and restart your local node to see the change take effect.

## Included policies

| Policy Set | Description |
|-----------|-------------|
| `classified_documents` | NATO clearance-level document filtering based on time windows |
| `patients` | Medical record transformation with blackening, deletion, and replacement |
| `demo_set` | Time-based permit/deny transitions with logging obligations, email notifications, and resource transformation |
| `argumentmodification` | Method argument modification via obligations |

All policies include streaming test scenarios demonstrating SAPL's reactive decision updates.

## Running tests locally

```
sapl test --dir ./policies --policy-hit-ratio 100
```

This discovers all `.sapl` and `.sapltest` files, runs the tests, and generates an HTML coverage report in `./sapl-coverage/html/`.

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
  integration.sapltest            End-to-end tests with pdp.json
  pdp.json                        PDP configuration (combining algorithm)
signing.pub                       Ed25519 public key (for verification)
.github/workflows/
  policy-pipeline.yml             CI/CD pipeline
```

## Pull request workflow

On pull requests, the pipeline runs tests only (no bundle publish, no Pages deploy). Reviewers see test results and coverage artifacts before merging policy changes.

## License

Apache License 2.0
