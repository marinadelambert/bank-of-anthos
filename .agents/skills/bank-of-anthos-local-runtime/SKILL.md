---
name: bank-of-anthos-local-runtime
description: Run real Bank of Anthos ledger containers and optional browser stack locally without GCP credentials, then verify money movement.
---

# Local runtime testing

Use isolated Docker names/network and loopback-bound host ports. Do not use production databases or keys.

## Devin Secrets Needed

None for local-only testing. `extras/jwt/jwt-secret.yaml` contains the intentionally public demo RSA keypair. Decode `data.jwtRS256.key` and `data.jwtRS256.key.pub` to temporary files and mount read-only. Never deploy these demo keys publicly.

## Build and start

- Confirm Docker access and the intended JDK with `docker info` and `java -version`.
- If already compiled, do not rerun verification solely to start containers.
- `mvn -B -pl src/ledger/ledgerwriter,src/ledger/balancereader,src/ledger/transactionhistory,src/ledgermonolith jib:dockerBuild` loads images as `<artifactId>:1.0` by default. Verify generated tags in output.
- Build `src/ledger/ledger-db` and optionally `src/accounts/accounts-db`, `src/frontend`, `src/accounts/userservice`, `src/accounts/contacts` with their Dockerfiles. Python images install their locked dependencies with uv; no host Python dependency install is needed.
- Use a user-defined Docker network and service aliases `ledger-db`, `accounts-db`, `balancereader`, `ledgerwriter`, `transactionhistory`, `userservice`, `contacts`.
- Ledger DB: `POSTGRES_DB=postgresdb`, `POSTGRES_USER=admin`, `POSTGRES_PASSWORD=password`, `LOCAL_ROUTING_NUM=883745000`. For a deterministic empty ledger set `USE_DEMO_DATA` to an **empty string**, not `False`: the current init script's conditional may otherwise seed transactions.
- Accounts DB: `POSTGRES_DB=accountsdb`, same demo DB user/password, `USE_DEMO_DATA=True`, `LOCAL_ROUTING_NUM=883745000`. This creates testuser/alice/bob/eve and contacts.

Java containers:

```text
PORT=8080
VERSION=local-runtime-test
LOCAL_ROUTING_NUM=883745000
PUB_KEY_PATH=/keys/public
SPRING_DATASOURCE_URL=jdbc:postgresql://ledger-db:5432/postgresdb
SPRING_DATASOURCE_USERNAME=admin
SPRING_DATASOURCE_PASSWORD=password
BALANCES_API_ADDR=balancereader:8080
ENABLE_TRACING=false
ENABLE_METRICS=false
SPRING_CLOUD_GCP_CORE_ENABLED=false
SPRING_CLOUD_GCP_TRACE_ENABLED=false
SPRING_CLOUD_GCP_METRICS_ENABLED=false
NAMESPACE=local
JAVA_TOOL_OPTIONS=-Xmx384m
```

Give each split ledger container an explicit hyphenated hostname, e.g. `--hostname ledgerwriter-local`. The custom Stackdriver resource-label code may parse the hostname before export is disabled, and default Docker ID hostnames can trigger `StringIndexOutOfBoundsException`. Disabling only `ENABLE_METRICS` may leave Spring GCP metrics auto-configuration enabled; disabling core without disabling GCP metrics can cause a missing `GcpProjectIdProvider` startup failure. Check startup logs, not only container running state.

Python containers need `PORT`, `VERSION`, `ENABLE_TRACING=false`, `PUB_KEY_PATH`, and `LOCAL_ROUTING_NUM`. Userservice additionally needs `PRIV_KEY_PATH`, `TOKEN_EXPIRY_SECONDS=3600`, and `ACCOUNTS_DB_URI=postgresql://admin:password@accounts-db:5432/accountsdb`. Contacts needs the same DB URI.

Frontend:

```text
BALANCES_API_ADDR=balancereader:8080
TRANSACTIONS_API_ADDR=ledgerwriter:8080
HISTORY_API_ADDR=transactionhistory:8080
USERSERVICE_API_ADDR=userservice:8080
CONTACTS_API_ADDR=contacts:8080
DEFAULT_USERNAME=testuser
DEFAULT_PASSWORD=bankofanthos
METADATA_SERVER=127.0.0.1
SCHEME=http
```

The metadata-server override avoids waiting on unavailable cloud metadata locally; metadata warnings are expected. Publish frontend port 8080 on a loopback host port. Demo login is `testuser` / `bankofanthos`.

## High-signal verification

- Run `docker exec <container> java -version` and read the application startup Java version; `/version` reports the configured service string, **not** the JVM version.
- All four services expose `/ready` and `/version`; `/healthy` exists on the two readers and monolith, not ledgerwriter.
- With empty ledger, sign in to `/login`, confirm $0.00, click **Deposit Funds**, keep **External Bank**, enter an exact amount, submit **Deposit**.
- Click **Send Payment**, select **Alice**, enter a smaller exact amount, submit **Send**. Assert both history rows and exact remaining balance after reload.
- For direct API checks, independently sign RS256 tokens with the demo private key and string `acct` claim plus `iat`/`exp`. Do not extract browser session cookies.
- POST `/transactions` uses JSON `fromAccountNum`, `fromRoutingNum`, `toAccountNum`, `toRoutingNum`, integer-cent `amount`, unique `uuid`. Expect 201 `ok`.
- Demo local routing is `883745000`, external account/routing `9099791699` / `808889588`, testuser account `1011226111`, Alice account `1033623433`.
- Reader caches poll asynchronously; bound any readback retry (e.g. 5 seconds). GET `/balances/{acct}` returns integer cents; `/transactions/{acct}` returns transaction JSON.
- Replay UUID returns 400 `duplicate transaction uuid`; invalid JWT/cross-account reads return 401 `not authorized`. Verify no extra DB rows.
- Monolith exposes the same write/balance/history endpoints and can read the same isolated ledger for parity checks.

Capture shell outputs for runtime/API proof and record only actual browser interactions. Keep setup retries out of the recording, but disclose them. Stop or remove only containers/network created by the testing session when finished, unless leaving them for review is requested.
