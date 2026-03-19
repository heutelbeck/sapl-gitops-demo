# GitOps for Authorization Policies

## What this scenario is about

Authorization policies change. Regulations tighten, roles evolve, new resources appear. The question is not whether policies will change, but how those changes reach your running systems safely.

This scenario demonstrates a GitOps pipeline for SAPL authorization policies: policies and tests live in Git, every change is tested and coverage-gated in CI, signed bundles are published as release artifacts, and running SAPL Nodes pick up new bundles automatically.

## The problem

Deploying authorization policy changes is uniquely risky. A misconfigured firewall rule is bad. A misconfigured authorization policy that silently permits access to patient records, financial data, or classified documents is worse, because it may go undetected until an audit or a breach.

Common failure modes:

- **Untested changes.** A policy author fixes one rule and unknowingly breaks another. Without automated tests, the regression ships to production.
- **No coverage visibility.** Even with tests, teams cannot tell which policy conditions are actually exercised. Dead branches hide latent bugs.
- **Manual deployment.** Copying policy files to servers introduces human error and makes rollbacks painful.
- **No integrity verification.** If a policy file is modified on a server (accidentally or maliciously), the PDP loads it without question.
- **No audit trail.** When something goes wrong, there is no record of which policy version was deployed, when, or by whom.

Git solves the audit trail. CI solves testing. Cryptographic signing solves integrity. Remote bundle polling solves deployment. This demo wires them together.

## How it works

```
git push --> CI pipeline --> GitHub Release --> SAPL Node
               |                  |                |
           test policies     publish signed    poll + verify
           coverage gates    bundle + key      load policies
               |                  |                |
           GitHub Pages      latest-bundle     serve decisions
           (coverage HTML)   (release tag)     (HTTP API)
```

The pipeline has three stages:

1. **Test.** The SAPL CLI discovers all `.sapl` policies and `.sapltest` test files, runs every scenario, and enforces coverage quality gates. If any test fails or coverage is below threshold, the pipeline stops. An HTML coverage report is deployed to GitHub Pages.

2. **Sign and publish.** The CLI packages all policies and `pdp.json` into a `.saplbundle` archive, signs it with an Ed25519 private key stored as a GitHub Secret, verifies the signature against the committed public key, and uploads the bundle to a GitHub Release.

3. **Consume.** A SAPL Node configured with `REMOTE_BUNDLES` polls the GitHub Release URL. When it detects a new bundle (via HTTP ETag), it downloads it, verifies the Ed25519 signature, and hot-reloads the policies. No restart, no manual intervention.

## The demo policies

This repository includes four policy sets from a reactive web application:

| Policy Set | What it demonstrates |
|-----------|---------------------|
| `classified_documents` | Time-window-based clearance levels (NATO_RESTRICTED, COSMIC_TOP_SECRET, NATO_UNCLASSIFIED) with obligation-driven document filtering |
| `patients` | Medical record transformation: blackening ICD codes, deleting diagnoses, replacing fields, with progressive disclosure over time |
| `demo_set` | Permit-to-deny transitions, resource transformation (wrapping responses), email obligations, logging advice |
| `argumentmodification` | Method argument modification via obligations |

All policies include streaming test scenarios that verify how decisions change as attribute values evolve over time.

## The scenario in action

### Testing catches mistakes

The test suite includes an integration test that loads all policies together with `pdp.json` through the combining algorithm. This catches configuration errors that unit tests miss:

```sapltest
requirement "combined policy evaluation with pdp.json" {
    given
        - configuration "."

    scenario "unmatched action denied by default"
        given
            - attribute "nowMock" <time.now> emits "t1"
            - function time.secondOf(any) maps to 5
        when "user" attempts { "java": { "name": "deleteEverything" } } on "resource"
        expect deny;
}
```

A broken `pdp.json` (wrong combining algorithm, invalid field) fails this test before the bundle is ever built.

### Coverage gates prevent blind spots

The pipeline enforces quality gates:

- 100% policy set hit ratio -- every policy set is evaluated by at least one test
- 100% policy hit ratio -- every individual policy is matched by at least one test
- 80% condition hit ratio -- most condition branches are exercised

If coverage drops below these thresholds, the pipeline fails with exit code 3 and no bundle is published.

### Signed bundles prevent tampering

Every bundle is signed with an Ed25519 key. The SAPL Node verifies the signature before loading. If someone modifies the bundle in transit or on the release, the node rejects it.

```
sapl bundle verify -b default.saplbundle -k signing.pub
```

### Remote polling closes the loop

A running SAPL Node polls the GitHub Release URL every 60 seconds. When the pipeline publishes a new bundle, the node detects the change via HTTP ETag, downloads the new bundle, verifies its signature, and hot-reloads the policies. The `configurationId` in each bundle includes the Git commit hash for traceability.

## Try it yourself

### Prerequisites

Download the SAPL CLI from [SAPL releases](https://github.com/heutelbeck/sapl-policy-engine/releases/tag/snapshot-node) and add it to your PATH.

### Fork and configure

1. **Fork** this repository on GitHub.

2. **Generate a signing keypair:**

```
sapl bundle keygen -o signing
```

3. **Add the private key as a GitHub Secret:**
   - Settings > Secrets and variables > Actions
   - Create secret `BUNDLE_SIGNING_KEY` with the contents of `signing.pem`

4. **Commit the public key** (if you generated a new keypair):

```
git add signing.pub
git commit -m "add signing public key"
git push
```

5. **Enable GitHub Pages:**
   - Settings > Pages > Source: **GitHub Actions**

6. **Trigger the pipeline:**
   - Actions > Policy Pipeline > Run workflow

After the run completes:

- **Coverage report** at `https://<you>.github.io/sapl-gitops-demo/`
- **Signed bundle** in Releases under `latest-bundle`
- **Test results and coverage metrics** in the Actions run summary

### Run tests locally

```
sapl test --dir ./policies --policy-hit-ratio 100
```

### Run a SAPL Node against the published bundle

#### One-time download

```
gh release download latest-bundle --pattern "*.saplbundle"
sapl bundle inspect -b default.saplbundle
sapl --bundle default.saplbundle --no-verify
```

#### Remote bundle polling (auto-updates)

Create a node directory:

```
mkdir -p sapl-node/config
gh release download latest-bundle --pattern "signing.pub" --dir sapl-node
```

Create `sapl-node/config/application.yml`:

```yaml
server:
  port: 8443
  ssl:
    enabled: false

io.sapl.pdp.embedded:
  pdp-config-type: REMOTE_BUNDLES

  remote-bundles:
    base-url: https://github.com/<you>/sapl-gitops-demo/releases/download/latest-bundle
    pdp-ids:
      - default.saplbundle
    mode: POLLING
    poll-interval: 60s

  bundle-security:
    public-key-path: signing.pub
```

Start the node:

```
cd sapl-node && sapl
```

The node fetches the signed bundle, verifies its signature, and starts serving decisions on `http://localhost:8443`. Push a policy change to `main` and watch the node reload within 60 seconds.

#### Query the node

```
sapl decide-once --remote --url http://localhost:8443 -s '{"role":"admin"}' -a '{"java":{"name":"getDocuments"}}' -r '"resource"'
```

### Make a policy change

Edit a policy under `policies/`, push to `main`, and watch the pipeline:

1. Tests run and coverage is verified
2. A new signed bundle is published
3. The running node picks it up automatically

Pull requests run tests without publishing, so reviewers see results before merging.

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

## Related

- [SAPL documentation](https://sapl.io/docs) -- language reference, testing DSL, node configuration
- [SAPL Testing](https://sapl.io/docs/latest/5_0_TestingSAPLPolicies) -- test DSL reference with mocking, streaming, and coverage
- [Remote Bundles](https://sapl.io/docs/latest/7_4_RemoteBundles) -- configuring SAPL Node for remote bundle polling
- [setup-sapl GitHub Action](https://github.com/heutelbeck/setup-sapl) -- install the SAPL CLI in GitHub Actions workflows
- [Spring Security scenario](/scenarios/spring/) -- securing a Spring Boot application with SAPL
- [AI scenarios](/scenarios/ai-rag/) -- authorization for RAG pipelines, tool calling, and human-in-the-loop

## License

Apache License 2.0
