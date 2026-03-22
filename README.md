# Ship Policies Like You Ship Code

Version, test, sign, and deploy authorization policies using Git, your CI system, and the monitoring stack you already run.

<p align="center"><img src="assets/pipeline.svg" alt="Policy pipeline: git push to CI to GitHub Release to SAPL Node" width="800"></p>

**[Full scenario walkthrough on sapl.io](https://sapl.io/scenarios/policy-ops/)**

## Quick start

1. **Fork** this repository
2. **Install the SAPL CLI** from [releases](https://github.com/heutelbeck/sapl-policy-engine/releases/tag/snapshot-node)
3. **Generate a signing keypair:** `sapl bundle keygen -o signing`
4. **Add secret** `BUNDLE_SIGNING_KEY` (contents of `signing.pem`) in Settings > Secrets > Actions
5. **Enable GitHub Pages** in Settings > Pages > Source: GitHub Actions
6. **Commit the public key:** `git add signing.pub && git commit -m "add signing key" && git push`
7. **Trigger the pipeline:** Actions > Policy Pipeline > Run workflow

After the run:
- [Coverage report](https://heutelbeck.github.io/sapl-gitops-demo/) at `https://<you>.github.io/sapl-gitops-demo/`
- Signed bundle in Releases under `latest-bundle`
- Test results in the Actions run summary

## Run a local SAPL Node

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
  print-text-report: true

  remote-bundles:
    base-url: https://github.com/<you>/sapl-gitops-demo/releases/download/latest-bundle
    pdp-ids:
      - default.saplbundle
    mode: POLLING
    poll-interval: 5s

  bundle-security:
    public-key-path: signing.pub
```

This uses `POLLING` mode (interval-based with ETag change detection), which works with any static HTTP server. For bundle servers that support holding connections, `LONG_POLL` mode delivers near-instant updates. See [Remote Bundles](https://sapl.io/docs/latest/7_4_RemoteBundles/) for configuration details.

```
cd sapl-node && sapl
```

## Try it

In a second terminal, subscribe to streaming decisions:

```
sapl decide --remote --url http://localhost:8443 -s '"user"' -a '{"http":{"contextPath":"/string"}}' -r '"resource"'
```

Change the suffix in `policies/argumentmodification.sapl`, update the matching value in `policies/argumentmodification.sapltest`, push to main, and watch the streaming output change after the pipeline publishes the new bundle.

## Run tests locally

```
sapl test --dir ./policies --policy-hit-ratio 100
```

See the [full scenario walkthrough](https://sapl.io/scenarios/policy-ops/) for details on each stage, the testing DSL, bundle signing, observability integration, and the LSP-based authoring experience.

## License

Apache License 2.0
