# Authentication Setup

By default the herald runs without authentication — suitable for trusted network environments. When `JWT_SECRET` is set, the herald enforces authentication on all OTLP gRPC connections and HTTP API calls.

## How it works

Instruments authenticate once at startup: they call `POST /auth/token` with their credential, receive a short-lived JWT (24h), and attach it to all subsequent gRPC calls. The client library handles this transparently.

```
Instrument              Herald
    │                      │
    │  POST /auth/token     │
    │  {credential, id} ──► │  validate credential
    │                       │  issue JWT (24h, HS256)
    │  ◄── {token, 86400}   │
    │                       │
    │  OTLP + Bearer token ►│  verify JWT on every span
```

## Enabling auth

Set `JWT_SECRET` in `deploy/.env`:

```bash
JWT_SECRET=your-random-signing-secret
```

Generate a strong secret:

```bash
openssl rand -hex 32
```

With `JWT_SECRET` set, any OTLP connection without a valid JWT is rejected. Instrument teams must update their pipelines before you enable enforcement — coordinate the rollout:

1. Deploy herald with `JWT_SECRET` unset (no enforcement).
2. Distribute credentials to instrument teams (see below).
3. Instrument teams update their pipelines to call `/auth/token`.
4. Set `JWT_SECRET` — enforcement begins.

## Key rotation (zero downtime)

`JWT_SECRET` accepts a comma-separated list:

```bash
JWT_SECRET=new-key,old-key
```

The **first** key signs new tokens; **all** keys are tried for verification. Rotate by prepending the new key, waiting 24h for old tokens to expire, then removing the old key.

---

## Auth backends

Each instrument independently configures how its credential is validated. Set the backend in the instrument YAML file:

### Secret backend

The instrument presents a registration secret. The herald compares the SHA-256 hash — the plaintext is never stored.

```yaml
instrument_id: MY_INST
auth:
  type: secret
  api_key_hash: "sha256:<hex-hash-of-the-secret>"
```

**Setup:**

```bash
# 1. Generate a secret — share this with the instrument team out-of-band
openssl rand -hex 32
# → e.g. a3f8c2...

# 2. Compute the hash — this goes in the YAML (safe to commit)
echo -n "a3f8c2..." | sha256sum
# → sha256:<hash>
```

The instrument team sets their secret as an environment variable and passes it to `setup()`:

```python
tel = setup(
    "my-pipeline",
    instrument_id="MY_INST",
    credential=os.environ["MY_INST_SECRET"],
    auth_endpoint="https://helixobs.example.org/auth/token",
)
```

### Token introspection backend

The instrument presents a token it already holds (e.g. from its own authentication system). The herald validates it by calling a remote `/verify` endpoint — HTTP 200 = valid.

```yaml
instrument_id: MY_INST
auth:
  type: token_introspection
  verify_url: https://auth.example.org/api/verify
```

The instrument passes a callable that returns the current token:

```python
tel = setup(
    "my-pipeline",
    instrument_id="MY_INST",
    credential=lambda: my_auth_system.get_token(),
    auth_endpoint="https://helixobs.example.org/auth/token",
)
```

The callable is invoked fresh on each HelixObs JWT refresh so short-lived upstream tokens are always current.

---

## Sharing credentials with instrument teams

Give each instrument team:

1. The herald auth endpoint: `https://helixobs.example.org/auth/token`
2. Their `instrument_id`
3. Their registration secret (secret backend) or the `verify_url` they should configure (introspection backend)

Never commit plaintext secrets. Only the SHA-256 hash goes in the instrument YAML.
