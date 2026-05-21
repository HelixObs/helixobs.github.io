# Adding an Instrument

Each instrument (telescope, detector, processing cluster) that connects to HelixObs needs an instrument YAML config file and, if auth is enabled, a set of credentials.

## 1. Choose an instrument ID

Pick a short, uppercase, stable string — e.g. `MY_INST`, `TELESCOPE_A`. This ID:

- Appears on every span, log line, and Prometheus label
- Scopes notification rules and silence rules
- Is used by Sherlock to load instrument context
- **Cannot be changed without breaking Grafana queries and notification routing**

Agree it with the instrument team before onboarding.

## 2. Create the instrument YAML

Create `deploy/instruments/my-inst.yml`:

```yaml
instrument_id: MY_INST

# Optional: AI troubleshooting context for Sherlock
description: |
  My instrument — a brief description of what it does and what its
  pipeline stages are.

# Optional: known error patterns and their usual causes
known_issues:
  - pattern: "connection refused"
    cause: "Upstream data service is down. Check the service status page."
  - pattern: "timeout"
    cause: "Network congestion or overloaded processing node."

# Optional: Prometheus metric names useful for diagnosing errors
metrics:
  - name: my_inst_queue_depth
    description: "Number of items waiting to be processed"
  - name: my_inst_processing_latency_seconds
    description: "End-to-end processing latency"

notifications:
  slack_webhook_env: MY_INST_SLACK_WEBHOOK
  github_token_env:  MY_INST_GITHUB_TOKEN

  events:
    helix.error:
      slack:
        channel: "#my-instrument-alerts"
        sample_window_seconds: 600
        max_per_window: 1
      github:
        repo: my-org/my-instrument
        labels: [helixobs, bug]
        auto_close_after_days: 7
        on_recurrence_after_close: reopen

# Optional: auth backend (only needed if JWT_SECRET is set on the herald)
auth:
  type: secret
  api_key_hash: "sha256:<hash>"   # see step 3
```

## 3. Set up auth (if enabled)

If `JWT_SECRET` is set on the herald, generate a credential for the instrument team:

```bash
# Generate a secret — share this with the instrument team out-of-band
openssl rand -hex 32
# → e.g. a3f8c2d1...

# Compute the hash — put this in the YAML
echo -n "a3f8c2d1..." | sha256sum
```

Add the hash to the YAML:

```yaml
auth:
  type: secret
  api_key_hash: "sha256:abc123..."
```

Share with the instrument team:
- The plaintext secret (out-of-band — never in the YAML or git)
- The auth endpoint: `https://helixobs.example.org/auth/token`
- Their `instrument_id`

## 4. Set notification credentials

Add the Slack webhook and GitHub token to `deploy/.env`:

```bash
MY_INST_SLACK_WEBHOOK=https://hooks.slack.com/services/...
MY_INST_GITHUB_TOKEN=ghp_...
```

The herald hot-reloads config files every 60 seconds — no restart needed after adding the YAML. Credential env vars require a herald restart to take effect (they are read at load time):

```bash
docker compose up -d herald
```

## 5. Configure log collection

Decide which log delivery path the instrument will use and communicate it to the instrument team:

| Path | What the instrument does | What you need |
|---|---|---|
| Sidecar (Alloy) | `otlp=False` (default) | Alloy running with the [required pipeline](log-collection.md) alongside their containers |
| OTLP | `otlp=True` | OTel Collector reachable at `:4319` from the instrument host |

## 6. Share connection details with the instrument team

Give them:

- Herald gRPC address: `helixobs.example.org:4317`
- Their `instrument_id`
- Auth credential (if auth enabled): plaintext secret + auth endpoint
- Log delivery path decision: sidecar or OTLP, and the relevant endpoint
- Grafana URL: `https://helixobs.example.org:3001`

## Onboarding checklist

- [ ] Instrument YAML created in `deploy/instruments/`
- [ ] Notification env vars set in `deploy/.env`
- [ ] Auth credential generated and shared (if auth enabled)
- [ ] Log delivery path agreed and configured
- [ ] Instrument team has herald address, instrument ID, and Grafana URL
- [ ] Test entity visible in Entity Inspector after first pipeline run
- [ ] `helix_instrument_id="MY_INST"` returns results in Loki
- [ ] Error Entities dashboard shows instrument in the dropdown
