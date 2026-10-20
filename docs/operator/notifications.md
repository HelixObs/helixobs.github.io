# Notifications

The herald dispatches Slack messages and GitHub issues automatically when a `helix.error` event (or any configured `helix.*` event) arrives. Configuration is per-instrument via YAML files hot-reloaded from the `instruments/` directory.

## Instrument config file

Create one YAML file per instrument in `deploy/instruments/`:

```yaml
instrument_id: MY_INST

notifications:
  # Environment variable that holds the Slack webhook URL for this instrument.
  slack_webhook_env: MY_INST_SLACK_WEBHOOK
  # Environment variable that holds the GitHub personal access token.
  github_token_env:  MY_INST_GITHUB_TOKEN
  # Optional: each alert links to <playbook_base>/<operation>. See Playbooks below.
  playbook_base: https://github.com/my-org/my-instrument/tree/main/runbooks

  events:
    helix.error:
      slack:
        channel: "#my-instrument-alerts"
        sample_window_seconds: 600   # rate-limit window
        max_per_window: 1            # max Slack messages per window
      github:
        repo: my-org/my-instrument
        labels: [helixobs, bug]
        on_recurrence_after_close: reopen   # or "new_issue"
```

Set the referenced environment variables in `deploy/.env`:

```bash
MY_INST_SLACK_WEBHOOK=https://hooks.slack.com/services/...
MY_INST_GITHUB_TOKEN=ghp_...
```

Config files are reloaded every 60 seconds — no herald restart needed for config changes.

## Fingerprinting and deduplication

Every error is fingerprinted from:

```
instrument_id | event_type | normalised_message | operation | stage
```

`operation` is the name of the span that emitted the error (the name passed to
`tel.operate(...)` or `tel.create(...)`), so the same message from two
operations is two error classes. `stage` is the event's `stage` metadata, if
the pipeline sets it.

The message is normalised before hashing — UUIDs, integers, IP addresses, and file paths are replaced with `?` — so the same error class affecting different entities produces the same fingerprint.

A single GitHub issue is maintained per fingerprint. When the same error recurs:

- **Issue open:** body is updated with the latest entity ID and occurrence count. No new comments.
- **Issue closed (`reopen`):** issue is reopened with a state-change comment, then body updated.
- **Issue closed (`new_issue`):** old DB record deleted, new issue created.

## Slack rate limiting

Within each `sample_window_seconds` window, at most `max_per_window` Slack messages are sent. Additional occurrences within the window are suppressed and reported as a digest at the window end:

```
1 more occurrence(s) suppressed in the last 600s.
Last: <error message>
Entity: <entity-id>
```

## Silence rules

Suppress notifications without changing config:

```bash
# Silence all notifications for an instrument for 4 hours
curl -X POST http://localhost:8080/api/v1/silences \
  -H "Content-Type: application/json" \
  -d '{"instrument_id": "MY_INST", "duration_seconds": 14400}'

# Silence a specific error fingerprint
curl -X POST http://localhost:8080/api/v1/silences \
  -d '{"fingerprint": "eb248a83", "duration_seconds": 3600}'

# List active silences
curl http://localhost:8080/api/v1/silences?instrument_id=MY_INST

# Delete a silence
curl -X DELETE http://localhost:8080/api/v1/silences/<id>
```

## Notification message format

Each Slack message and GitHub issue body includes:

- Error summary (normalised message)
- Operation name, and stage if the pipeline sets one
- Playbook link, if configured
- Entity ID (with link to Entity Inspector)
- First seen / last seen timestamps
- Total occurrence count
- Up to 10 recent entity IDs
- Link to the Error Entities Grafana dashboard

## Playbooks

Set `notifications.playbook_base` and each alert links to
`<playbook_base>/<operation>`, as a **Playbook** button in Slack and a line
near the top of the GitHub issue. Name each runbook folder or wiki page exactly
after its operation; lowercase kebab-case names work unchanged in URLs.

HelixObs never fetches the link. If no runbook exists for an operation yet, the
link is a 404, and the alert says to ask the operation's owner to add one.

Resolution, most specific first:

1. The matching override's `playbook`
2. `playbook_base/<operation>`
3. The event's `playbook`
4. `notifications.playbook`

Invalid URLs (not absolute `http`/`https`) are dropped with a warning; the rest
of the config still loads.

## Overrides

`overrides` under an event changes its settings for specific operations:

```yaml
    helix.error:
      overrides:
        - match: { operation: convert-data-to-hdf5 }
          slack:
            channel: "#baseband-alerts"
            webhook_env: BASEBAND_SLACK_WEBHOOK
            sample_window_seconds: 600
            max_per_window: 1
          github: off
        - match: { operation: baseband-callback, stage: request-to-kko-coco }
          playbook: https://wiki.example/kko
```

- `match.operation` is required, compared exactly with the operation name.
- `match.stage` is optional; when set, the event's `stage` metadata must match too.
- The first matching override wins. Unset `slack`, `github` and `playbook` are
  inherited from the event; `off` turns a destination off.
- The old `routes:` key is ignored, with a warning.
- An override whose operation has no events for 7 days logs a warning, since a
  renamed operation silently stops matching.

## Custom event notifications

Any `helix.*` span event can trigger notifications, not just `helix.error`. Add additional event keys under `notifications.events`:

```yaml
events:
  helix.error:
    slack: { ... }
    github: { ... }
  helix.event.quality-flag:
    slack:
      channel: "#science-alerts"
      sample_window_seconds: 3600
      max_per_window: 5
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| No Slack messages | Webhook env var not set or wrong variable name in YAML |
| No GitHub issues | Token env var missing, or token lacks `repo` scope |
| Duplicate issues | See [dedup notes above](#fingerprinting-and-deduplication); may indicate herald was restarted between GitHub API call and DB write |
| Notifications for known-noisy errors | Create a silence rule via the API |
