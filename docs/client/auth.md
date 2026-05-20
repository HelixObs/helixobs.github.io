# Authentication

By default, the HelixObs gateway runs without authentication — useful for local development. In production, the gateway can be configured to require a short-lived JWT before accepting OTLP spans. The client library handles token exchange transparently.

## When you need this

Your operator will tell you if authentication is required. If `JWT_SECRET` is not set on the gateway, auth is disabled and no credential is needed.

## Providing a credential

Pass `credential` and `auth_endpoint` to `setup()`:

=== "Static secret"

    ```python
    tel = setup(
        "my-pipeline",
        instrument_id="MY_INST",
        endpoint="gateway:4317",
        credential="my-registration-secret",
        auth_endpoint="https://gateway.example.org/auth/token",
    )
    ```

    The library exchanges your secret for a short-lived JWT at startup. The JWT is refreshed automatically before it expires — your pipeline never needs to handle token lifecycle.

=== "Callable (dynamic secret)"

    If your pipeline already has its own authentication system that issues short-lived tokens, pass a callable that returns the current credential:

    ```python
    def get_token() -> str:
        return my_auth_system.get_current_token()

    tel = setup(
        "my-pipeline",
        instrument_id="MY_INST",
        endpoint="gateway:4317",
        credential=get_token,           # called fresh on each token refresh
        auth_endpoint="https://gateway.example.org/auth/token",
    )
    ```

    The callable is invoked once at startup and again whenever the HelixObs JWT nears expiry (~1 hour before). This ensures that short-lived upstream credentials are always fresh when exchanged.

## What happens at startup

1. The library calls `POST /auth/token` with your credential and `instrument_id`.
2. The gateway validates the credential against its configured backend (shared secret hash or remote token introspection).
3. On success, the gateway returns a JWT valid for 24 hours.
4. The JWT is attached to all subsequent OTLP gRPC calls as `Authorization: Bearer <token>`.
5. The library refreshes the JWT automatically when fewer than 1 hour remains.

## Fail-fast behaviour

If the auth endpoint is unreachable or the credential is rejected, `setup()` raises immediately — your pipeline will not start in a misconfigured state.

## Credential security

- Never hard-code credentials in source code. Use environment variables or a secrets manager.
- The registration secret is hashed (SHA-256) on the gateway side — the plaintext is never stored.
- Share your registration secret with the operator out-of-band; only the hash goes into the instrument config file.

```python
import os

tel = setup(
    "my-pipeline",
    instrument_id="MY_INST",
    endpoint="gateway:4317",
    credential=os.environ["HELIXOBS_SECRET"],
    auth_endpoint=os.environ["HELIXOBS_AUTH_ENDPOINT"],
)
```
