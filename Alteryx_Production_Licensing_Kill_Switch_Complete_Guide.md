# Alteryx ETL Rationalisation

# Production Licensing, Kill-Switch, Usage Limits, Azure Key Vault and Databricks Integration Guide

**Document type:** Technical architecture and operational runbook\
**Audience:** Engineering, platform operations, security, solution
owners, and management\
**Application:** Alteryx ETL Rationalisation / AWA application\
**Runtime platform:** Azure Databricks Apps\
**License mechanism:** Signed JSON license artifact, Ed25519 signature
verification, startup-only enforcement\
**Status:** Reflects the implemented architecture and reported
deployment/test results available at the time this guide was prepared.

------------------------------------------------------------------------

## Document control and interpretation

This guide consolidates the previous date-based license enforcement
design and the subsequent addition of **Volume** and **Token Usage**
criteria. It also documents the separation of signing and distribution
secrets, how Azure Key Vault connects to Databricks, runtime behavior,
testing, operational procedures, and how the licensing package can be
adapted for another application.

The terms **implemented**, **reported**, and **recommended** are used
deliberately:

-   **Implemented** describes architecture or behavior established in
    the project context.
-   **Reported** describes a test or deployment result reported during
    implementation. Preserve the actual test output or deployment logs
    as the formal evidence record.
-   **Recommended** describes operational hardening or a reusable
    integration pattern; it should not be mistaken for a claim that a
    separate control has already been implemented.

This document does not contain any private key or LLM credential. Never
add secret values to this document.

------------------------------------------------------------------------

# 1. Executive summary

The application uses a **cryptographically signed license artifact** to
decide whether it is permitted to complete startup. The original
enforcement criterion was **Date**. The licensing package was then
extended to support three independently configurable criteria:

1.  **Date** --- allows startup only while the signed license remains
    within its validity window.
2.  **Volume** --- allows startup only while the cumulative volume value
    read from a configured usage data source remains below the signed
    limit.
3.  **Token Usage** --- allows startup only while the cumulative
    token-usage value read from a configured usage data source remains
    below the signed limit.

Each criterion has an enable/disable flag in the signed policy. The
policy is part of the signed artifact, so changing an enable flag or a
limit without the signing key invalidates the signature. A signed policy
must enable at least one criterion; a policy with all three disabled is
rejected at the authoring/signing boundary.

The production enforcement check runs **once during application
startup**. It is not a request-by-request check and does not run a
periodic heartbeat. If a criterion is enabled and its validation fails,
the application fails closed and startup is blocked.

The Volume and Token Usage features are **read-only consumption
controls**. The package reads the current cumulative values from a
configured usage source. It does not count business activity, record
token use, increment counters, write usage data, scan business datasets,
or integrate with an LLM provider SDK to calculate token usage. The
application owner supplies the usage database/source and ensures that it
contains authoritative counters.

The production signing private key and the signed license artifact are
stored separately:

-   Azure Key Vault **`alteryx-licensing`** stores the Ed25519 private
    signing key in secret **`alteryx-license-private-key`**. This is the
    signing-side vault and must not be accessible to the production
    application.
-   Azure Key Vault **`alteryx-licenseArtifacts`** stores the signed
    license JSON in secret **`alteryx-license`**. This is the
    distribution-side vault used by the production runtime.

Databricks has an Azure Key Vault-backed secret scope named
`alteryx-licenseArtifacts`, and the Databricks App has a Secret resource
whose resource key is `license-artifact`. `app.yaml` maps that resource
to the environment variable `ALTERYX_LICENSE_ARTIFACT`. The Databricks
App reads the injected environment variable; it does **not** need
notebook `dbutils` to retrieve the production license.

The core licensing package is provider-agnostic. Databricks-specific
access is an adapter at the boundary, rather than a dependency of the
core policy logic. This permits the same licensing concepts to be
embedded in another Python application by supplying the relevant secret
and usage-data adapters.

## 1.1 The runtime decision

``` text
Databricks App process starts
          |
          v
LicenseManager.validate_or_raise()
          |
          v
Retrieve signed artifact
          |
          v
Parse JSON and validate schema
          |
          v
Verify Ed25519 signature over canonical payload
          |
          v
Validate license identity (ID / product / environment)
          |
          v
Evaluate enabled criteria:
  - Date
  - Volume
  - Token Usage
          |
          v
Every enabled criterion passes?
       /       \
     YES        NO
      |          |
      v          v
Startup      Fail closed:
continues    startup blocked
```

**Important:** Because checks are startup-only, the system does not
guarantee that a process already running will be terminated at the exact
moment a date expires or a usage threshold is crossed. Enforcement takes
place on the next application startup/restart. If immediate mid-session
enforcement is required, that would be a separate architectural
requirement and must not be implied by this implementation.

------------------------------------------------------------------------

# 2. Requirements and design goals

The implementation addresses the following requirements.

## 2.1 License issuance must be controlled

Only an authorized signing process holding the Ed25519 private key can
create a valid license. The production app receives the signed artifact
and public verification key, not the private key.

## 2.2 License policy must be tamper-evident

The policy---including enabled criteria and their limits---is included
in the signed payload. A user cannot legitimately extend an expiry date,
raise a volume limit, raise a token limit, or disable an enforced
criterion merely by editing JSON. Any modification to a signed field
changes the canonical bytes and causes signature verification to fail.

## 2.3 Criteria must be independently configurable

The policy supports Date, Volume, and Token Usage independently. This
permits date-only licensing, usage-only licensing, or combinations such
as Date + Volume, Date + Token Usage, Volume + Token Usage, or all
three.

## 2.4 Usage values must be supplied by the application owner

The package is not a usage-metering system. It reads counters from a
configured source. This is intentional: metering logic is
application-specific, and a reusable licensing package should not assume
a database schema or intercept business operations.

## 2.5 Runtime checks must be startup-only

The app validates the signed artifact during startup. No renewal loop,
lease polling, periodic licensing check, or request-time license check
is part of the implemented design.

## 2.6 Core licensing must remain portable

The licensing core must not depend on Databricks, Azure Key Vault SDKs,
notebook `dbutils`, a specific SQL engine, or an LLM provider.
Platform-specific implementations are adapters injected by the
application.

## 2.7 Invalid states must fail closed

Missing or malformed artifacts, invalid signatures, identity mismatches,
expiry, or invalid usage-source data must not be treated as
authorization to start.

------------------------------------------------------------------------

# 3. Terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  License artifact                    JSON document containing license
                                      identity, validity information,
                                      feature metadata, policy, and
                                      signature.

  License payload                     The artifact fields covered by the
                                      signature, excluding the signature
                                      field itself.

  License policy                      Signed enforcement configuration:
                                      enabled criteria and relevant
                                      limits.

  Signer / issuer                     Authorized process that reads the
                                      private key and signs a payload.

  Runtime verifier                    Production application code that
                                      validates the artifact with the
                                      public key.

  Secret provider                     Adapter that retrieves the signed
                                      artifact from the runtime's
                                      configured secret delivery
                                      mechanism.

  Usage data source                   Read-only abstraction used to
                                      obtain current Volume and Token
                                      Usage counters.

  Startup kill switch                 Refusal to complete application
                                      startup when license validation
                                      fails.

  Fail closed                         An error or ambiguous state denies
                                      startup rather than permitting it.

  Databricks App resource key         Name used in `app.yaml` to
                                      reference an App resource,
                                      e.g. `license-artifact`.

  Environment variable                Runtime variable populated by
                                      Databricks from the configured App
                                      resource,
                                      e.g. `ALTERYX_LICENSE_ARTIFACT`.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 4. High-level architecture and trust boundaries

There are two distinct flows: **license issuance** and **license
consumption**.

## 4.1 Issuance and distribution

``` text
Authorized signing environment
  |
  | Retrieves private key from the restricted signing-side vault/scope
  v
Ed25519 private key
  |
  | Signs canonical JSON payload
  v
Signed license artifact
  |
  | Authorized publication operation
  v
Azure Key Vault: alteryx-licenseArtifacts
  Secret: alteryx-license
  |
  | Azure Key Vault-backed Databricks secret scope
  v
Databricks secret scope: alteryx-licenseArtifacts
  |
  | Databricks App Secret resource (resource key: license-artifact)
  v
Environment variable: ALTERYX_LICENSE_ARTIFACT
```

## 4.2 Runtime validation

``` text
Databricks App startup
  |
  v
LicenseManager
  |-- reads ALTERYX_LICENSE_ARTIFACT via injected provider
  |-- parses JSON and validates model/schema
  |-- verifies signature using embedded Ed25519 public key
  |-- checks identity binding
  |-- evaluates enabled policy criteria
  |     |-- Date: signed time window
  |     |-- Volume: current counter from UsageDataSource
  |     `-- Token Usage: current counter from UsageDataSource
  v
Startup permitted only if every enabled criterion passes
```

## 4.3 Trust boundaries

-   The **private signing key** belongs only to the authorized signing
    boundary.
-   The **signed artifact** may be delivered to the production runtime
    because it does not provide signing authority.
-   The **public key** is safe to distribute with the verifier. It
    verifies signatures but cannot generate them.
-   The **usage database** is a separate trust boundary. Its integrity
    and correctness are necessary for reliable Volume/Token Usage
    enforcement.
-   The **Databricks App environment variable** is a secret-delivery
    mechanism, not the signing authority.
-   The **license core** should not be allowed to silently replace
    signed limits with local developer configuration.

------------------------------------------------------------------------

# 5. Azure Key Vaults: purpose and communication paths

Two vaults are used deliberately. They are not interchangeable.

## 5.1 `alteryx-licensing` --- signing-side vault

**Secret name:** `alteryx-license-private-key`

This vault stores the Ed25519 private signing key. The key is used by an
authorized issuance process to sign a license payload. The production
Databricks App does not need this secret and must not be granted access
to it.

Protect this vault and secret as a high-value signing credential. Access
should be restricted to the identities and operators responsible for
issuing licenses. Never commit, print, log, package, or distribute the
private key.

The implementation history used a Databricks notebook as a controlled
signing environment and retrieved the private key from the
testing/signing secret scope `alteryx-licenseSigning`. The exact method
by which an authorized issuer obtains the key can vary by organization,
but the production app must never use the signing path.

## 5.2 `alteryx-licenseArtifacts` --- artifact distribution vault

**Secret name:** `alteryx-license`

This vault stores the signed JSON license artifact. It contains the
public payload and signature---not the private signing key.

The Databricks production App needs access to this artifact, not to the
signing vault. A new license is published by updating this secret with a
newly signed artifact. The next application startup reads and validates
the new value.

## 5.3 Do the two vaults communicate with each other?

**No direct vault-to-vault communication is required by this design.**

The issuer reads the private key from the signing-side vault, signs the
payload in the controlled signing environment, and then publishes the
resulting signed JSON to the artifact vault. That publication is an
explicit operation performed by an authorized process/operator. It is
not a live synchronization between the two Key Vaults.

``` text
alteryx-licensing
  (private key)
       |
       | authorized signer retrieves key
       v
Signing process (Python / cryptography)
       |
       | produces signed JSON
       v
Authorized publish operation
       |
       v
alteryx-licenseArtifacts
  (signed artifact)
```

The production App only follows the artifact-distribution path. It does
not contact the signing vault and does not ask the signing vault to sign
anything.

## 5.4 Azure CLI commands used for the artifact vault

The following commands reflect the operational flow used in this
project. Run them only under an Azure identity authorized for the target
vault.

Check artifact-secret metadata without displaying the secret value:

``` bash
az keyvault secret show \
  --vault-name alteryx-licenseArtifacts \
  --name alteryx-license \
  --query "{name:name, enabled:attributes.enabled}" \
  -o json
```

Publish a signed JSON artifact from Azure Cloud Shell without echoing
the pasted value:

``` bash
read -s -p "Paste signed license JSON: " LICENSE_JSON
echo

az keyvault secret set \
  --vault-name alteryx-licenseArtifacts \
  --name alteryx-license \
  --value "$LICENSE_JSON" \
  --query "name" \
  -o tsv

unset LICENSE_JSON
```

The command prints the secret name rather than the secret value. Avoid
`echo "$LICENSE_JSON"`, shell tracing (`set -x`), screenshots of the
entered value, or command histories that contain the JSON.

### Operational note about backups

A rollback copy can be retained in a controlled operator session or,
preferably, through the organization's approved
secret/version-management process. Do not assume a shell variable is a
durable backup: it disappears when the shell session ends. The previous
guide used a variable such as `VALID_LICENSE`; that is useful for an
immediate same-session rollback only.

------------------------------------------------------------------------

# 6. Databricks integration: secret scope, App resource, and `app.yaml`

Azure Key Vault stores the artifact, but the production app needs a
supported mechanism to receive it at runtime.

## 6.1 Databricks secret scope

The production secret scope is:

``` text
Scope: alteryx-licenseArtifacts
Key:   alteryx-license
```

It is Azure Key Vault-backed by the Azure Key Vault named
`alteryx-licenseArtifacts`. The scope is a Databricks access layer for
the artifact; it does not contain the private signing key.

Other project scopes have separate purposes:

  ------------------------------------------------------------------------------------
  Scope                        Purpose                         Production App use
  ---------------------------- ------------------------------- -----------------------
  `alteryx-llm-secrets`        Existing LLM credentials,       Used by the LLM
                               including `azure-llm-key`       integration as
                                                               configured

  `alteryx-licenseArtifacts`   Signed license artifact         Yes; exposed through
                               `alteryx-license`               the App secret resource

  `alteryx-licenseSigning`     Signing/testing access to       **No. Never attach to
                               `alteryx-license-private-key`   the production App**
  ------------------------------------------------------------------------------------

A Databricks secret scope and an App resource are distinct objects.
Creating the scope does not, by itself, guarantee that a Databricks App
process has the secret in its environment.

## 6.2 Databricks App Secret resource

The App resource was configured with the following values:

  Field           Value
  --------------- ----------------------------
  Resource type   Secret
  Secret scope    `alteryx-licenseArtifacts`
  Secret key      `alteryx-license`
  Permission      `Can read`
  Resource key    `license-artifact`

The resource key is a Databricks App resource reference. It is not the
Azure Key Vault secret name and not the Databricks scope name.

## 6.3 `app.yaml`

The environment mapping is:

``` yaml
env:
  - name: ALTERYX_LICENSE_ARTIFACT
    valueFrom: license-artifact
```

This entry means: populate the application environment variable
`ALTERYX_LICENSE_ARTIFACT` from the App resource whose resource key is
`license-artifact`.

The full `app.yaml` may contain other existing variables and a command
section. Do not replace the whole file with this small fragment;
merge/retain the entry alongside the existing configuration. In
particular, preserve the application's current command and unrelated LLM
settings.

## 6.4 Runtime access: environment variable, not notebook `dbutils`

In the Databricks App runtime, the licensing adapter reads the injected
environment variable. The production path is conceptually:

``` python
import os

artifact_text = os.getenv("ALTERYX_LICENSE_ARTIFACT")
if not artifact_text:
    raise LicenseSecretError(
        "ALTERYX_LICENSE_ARTIFACT is missing or empty"
    )
```

This is the critical distinction from notebook execution. Notebook code
can use `dbutils.secrets.get(...)`, but `dbutils` is not a guaranteed
API inside a Databricks App process. During deployment, the initial
adapter attempted to resolve `dbutils` in the App runtime and startup
failed because `dbutils` was unavailable. The correction was to use the
App resource's environment-variable injection.

The adapter should fail closed when the injected value is missing or
empty. It must not silently substitute a local file, an unsigned
artifact, or a different secret.

## 6.5 Does Databricks call Azure Key Vault directly at every startup?

Not through the licensing code in the current runtime path. The
configured Databricks secret scope is Azure Key Vault-backed, and the
Databricks App resource injects the secret value into the process
environment. The application then reads that value with `os.getenv`. The
application code does not independently instantiate an Azure Key Vault
client or retrieve the private key.

The underlying platform manages the resource integration. Do not
describe this as direct application-level vault-to-vault communication.

## 6.6 Scope-level access caution

Databricks secret permissions and App resource permissions must be
reviewed carefully. Keep the production artifact scope narrow and do not
place unrelated credentials in a scope that the production App can read.
Never grant the production App access to `alteryx-licenseSigning`.

------------------------------------------------------------------------

# 7. Cryptographic design: Ed25519 and canonical JSON

## 7.1 Why asymmetric signatures?

The design uses Ed25519, an asymmetric digital-signature algorithm. The
signer holds the private key; the runtime holds only the public key.

-   The private key can sign a payload.
-   The public key can verify a signature.
-   The public key cannot be used to create a valid signature.

This separates **license issuance authority** from **license validation
authority**.

## 7.2 Public key

The public key embedded in the application is:

``` text
aykIwjC0U0mxmTXUDhQdwBCiogj8YRNWy/8EieAfx9s=
```

This is Base64-encoded raw Ed25519 public-key material. The
corresponding private key must not be embedded in the application,
included in the frontend, or shipped in the protected artifact.

## 7.3 Canonicalization

Digital signatures cover bytes, not abstract Python dictionaries. The
issuer and verifier must serialize the payload into exactly the same
bytes.

The project uses deterministic JSON serialization:

``` python
canonical_payload = json.dumps(
    payload,
    sort_keys=True,
    separators=(",", ":"),
    default=str,
).encode("utf-8")
```

-   `sort_keys=True` provides deterministic object-key ordering.
-   `separators=(",", ":")` removes insignificant whitespace.
-   UTF-8 encoding turns the JSON text into the exact byte sequence that
    Ed25519 signs.

The verifier reconstructs the payload by excluding the `signature` field
and applying the same canonicalization. If any signed field
changes---including policy flags or usage limits---the reconstructed
bytes differ and verification fails.

**Important implementation rule:** Do not hand-edit the signed JSON
after signing. Even a semantically small change to a signed field
requires regenerating the signature with the authorized private key.

## 7.4 Signature size and Base64

An Ed25519 signature is 64 bytes. Base64 representation is 88 characters
for a 64-byte signature, including the usual trailing padding. This is a
useful diagnostic check, but length alone does not establish validity;
verification must succeed.

------------------------------------------------------------------------

# 8. License artifact and signed policy

The artifact carries identity, validity data, feature metadata, a signed
enforcement policy, and a signature.

The established identity is:

``` text
license_id:  CLIENT-ALTERYX-001
product:     alteryx-etl
environment: production
```

The current application is bound to this production identity. The
runtime does not permit local developer configuration to override the
authoritative signed identity/policy.

## 8.1 Legacy fields and the policy extension

The original artifact contained fields such as:

``` json
{
  "license_id": "CLIENT-ALTERYX-001",
  "product": "alteryx-etl",
  "environment": "production",
  "issued_at": "2026-10-06T00:00:00+00:00",
  "expires_at": "2026-11-06T00:00:00+00:00",
  "features": {
    "workflow_analysis": true,
    "portfolio_rationalisation": true,
    "python_translation": true,
    "export_reports": true
  },
  "signature": "<Base64 Ed25519 signature>"
}
```

The final policy-aware schema also requires a **`policy`** field. An
earlier deployment failed at startup with a Pydantic schema-validation
error because the artifact had no `policy` field. The correct fix was to
issue a new artifact that contains the mandatory signed policy, not to
make the policy optional at runtime.

The following is a **conceptual example** of the policy intent, not a
replacement for the exact Pydantic model in the deployed source:

``` json
{
  "policy": {
    "date": {
      "enabled": true
    },
    "volume": {
      "enabled": false,
      "limit": 9999
    },
    "token_usage": {
      "enabled": false,
      "limit": 500000
    }
  }
}
```

The actual model defines the exact fields and types. Keep the schema in
`backend/app/licensing/models.py` authoritative when generating an
artifact. The required runtime fields include
`artifact.policy.date.enabled`, `artifact.policy.volume.enabled`,
`artifact.policy.volume.limit`, `artifact.policy.token_usage.enabled`,
and `artifact.policy.token_usage.limit`. Date validity is also
represented by the license's issued/expiry data as defined by the
deployed models.

## 8.2 Why the policy is mandatory

Making `policy` mandatory provides one authoritative runtime policy. If
the field were optional, older artifacts could be interpreted under a
fallback policy, which risks ambiguity over which limits actually apply.
The runtime must not silently merge local developer flags with signed
artifact flags.

## 8.3 Policy integrity

The `policy` object is part of the canonical payload covered by the
signature. Therefore:

-   changing `date.enabled` invalidates the signature;
-   changing `volume.enabled` invalidates the signature;
-   changing `volume.limit` invalidates the signature;
-   changing `token_usage.enabled` invalidates the signature;
-   changing `token_usage.limit` invalidates the signature.

A new policy requires a new artifact signed by the authorized issuer.

## 8.4 Feature flags versus enforcement criteria

Do not confuse `features` with `policy`:

-   `features` describes which application capabilities are licensed,
    such as workflow analysis or report export.
-   `policy` defines the criteria that must pass before startup is
    allowed.

They are related licensing concepts but are not interchangeable. The
presence of a feature flag does not replace the Date, Volume, or Token
Usage policy checks.

------------------------------------------------------------------------

# 9. The three enforcement criteria

The three criteria are independent switches in the signed policy. The
runtime evaluates each enabled criterion. **All enabled criteria must
pass.** A disabled criterion does not block startup merely because its
usage value is over a configured limit; however, its policy fields still
need to satisfy the schema's validity rules.

## 9.1 Date criterion

### Purpose

Date licensing bounds the period during which the application may start.
It is suitable for evaluation periods, fixed-term client agreements,
time-limited pilots, and license renewals.

### Data and semantics

The artifact includes issued/expiry timestamps. The project uses
UTC-aware timestamps to avoid machine-local timezone ambiguity. The
expiry boundary is strict: when the current time is greater than or
equal to the expiry instant, the license is expired.

Conceptually:

``` python
if date_enabled and current_time_utc >= expires_at_utc:
    raise LicenseExpiredError(...)
```

This is an explanatory pseudocode fragment, not a replacement for the
implementation.

### Boundary behavior

-   Before expiry: Date criterion passes.
-   Exactly at expiry: Date criterion fails.
-   After expiry: Date criterion fails.

Tests have covered UTC `Z` timestamps, offset-aware timestamps,
UTC-naive timestamp handling, non-UTC offsets, and the expiry boundary
triad. Use the behavior in the actual deployed model/validator as the
authority.

### What it does not do

Date enforcement is checked at startup. It does not create a background
timer that kills a process at the exact expiry instant. A restart is
needed for the startup gate to run again.

## 9.2 Volume criterion

### Purpose

Volume licensing limits a cumulative business-defined quantity. The
exact meaning of "volume" belongs to the embedding application---for
example, a count of workflows or another agreed unit. The licensing core
should not assume that volume means rows, files, workflows, or any other
unit unless the host application defines it.

### Data source

The package reads the current volume counter from the configured usage
data source. The application owner supplies the source and ensures the
counter has the agreed semantics.

### Enforcement semantics

The signed policy provides an enable flag and an upper limit. The
current value is read from the source at startup. The implemented
boundary behavior is:

``` text
current volume < signed limit  -> criterion passes
current volume >= signed limit -> criterion fails
```

Therefore a limit of `100` allows startup when the current value is
`99`, but denies startup when it is `100` or greater.

This is a **cumulative threshold**. It is not a reservation system and
does not prevent two simultaneous processes from racing across a
threshold. Since the check is startup-only and read-only, the host
application remains responsible for how counters are maintained and how
concurrent usage is accounted for.

### No metering side effects

The licensing package does not: - increment the volume counter; - create
usage records; - update the database; - scan business data to calculate
a count; - infer volume from API requests; - reset counters; - write a
new counter value after a successful startup.

It only reads the authoritative current value and compares it with the
signed limit.

## 9.3 Token Usage criterion

### Purpose

Token Usage licensing limits a cumulative token counter supplied by the
host application. This is useful when an application consumes a
token-metered model or service and the commercial agreement needs a
defined token ceiling.

### Data source

The package reads the existing token-usage counter from the configured
usage source. It does not call Azure AI Foundry, OpenAI, or another LLM
provider to calculate usage, and it does not intercept LLM requests.

### Enforcement semantics

The signed policy supplies `enabled` and `limit`. The implemented
boundary behavior is:

``` text
current token usage < signed limit  -> criterion passes
current token usage >= signed limit -> criterion fails
```

For example, with a signed token limit of `500000`, a current value of
`499999` passes, while `500000` and `500001` fail.

The number must be the cumulative unit agreed by the host application.
If the database stores tokens, the package treats the value as tokens;
it does not convert requests, estimate tokens, or infer missing values.

### No LLM coupling

The licensing core must not import an LLM SDK or depend on Azure AI
Foundry. The host application can independently write token counts to
its usage store. The licensing package only reads the stored aggregate
through the `UsageDataSource` interface.

## 9.4 Combinations of enabled criteria

All three switches are independent. The supported combinations are:

  -----------------------------------------------------------------------------------
                Date             Volume        Token Usage Runtime behavior
  ------------------ ------------------ ------------------ --------------------------
                   0                  0                  0 Invalid signed policy;
                                                           reject at
                                                           authoring/signing/schema
                                                           validation.

                   1                  0                  0 Enforce date only.

                   0                  1                  0 Enforce volume only.

                   0                  0                  1 Enforce token usage only.

                   1                  1                  0 Date and volume must both
                                                           pass.

                   1                  0                  1 Date and token usage must
                                                           both pass.

                   0                  1                  1 Volume and token usage
                                                           must both pass.

                   1                  1                  1 Date, volume, and token
                                                           usage must all pass.
  -----------------------------------------------------------------------------------

Here, `1` means enabled and `0` means disabled.

The `0/0/0` case is intentionally a **policy-authoring error**, not an
error caused by the local developer configuration. Runtime enforcement
is based on the signed artifact. Local flags must not be used to turn
off all checks or override the signed policy.

------------------------------------------------------------------------

# 10. Usage data source: portable interface and operational contract

## 10.1 Why an interface is needed

A reusable licensing package cannot assume that every host application
uses the same database or schema. One application might store usage in
SQLite, another in PostgreSQL, and another in a platform-specific table
or service.

The core therefore uses a generic `UsageDataSource`
protocol/abstraction. A concrete adapter knows how to connect to the
host's usage store and read the current values. The licensing criteria
call the interface rather than importing a database-specific client.

Conceptually:

``` text
LicenseManager / criteria
          |
          v
UsageDataSource interface
          |
          +--> SQLite adapter (test-only example)
          +--> PostgreSQL adapter (host-supplied, if needed)
          +--> Databricks adapter
          `--> another application-specific adapter
```

The above lists possible adapters; it does not claim that each one has
been implemented in the project. Databricks is supported through an
adapter, while the usage test bundle used a test-only SQLite source.

## 10.2 Read-only contract

The source should expose current cumulative values, not perform
metering. The conceptual interface is:

``` python
from typing import Protocol

class UsageDataSource(Protocol):
    def get_current_volume(self) -> int:
        ...

    def get_current_token_usage(self) -> int:
        ...
```

This illustrates the interface contract. Preserve the actual method
names and signatures in `backend/app/licensing/usage_source.py`; do not
replace the deployed implementation merely to match this illustrative
snippet.

The source must return a valid, non-negative value in the agreed unit.
Missing rows, malformed values, negative values, connection failures, or
unavailable sources must fail closed whenever the corresponding
criterion is enabled.

## 10.3 Consumer-only semantics

The package consumes the values. It does not own their lifecycle. The
host application/database owner must define: - what a unit of volume
means; - what token count is authoritative; - whether counters are
cumulative across the license term or reset at a billing boundary; - how
and when the application updates the source; - how concurrent operations
are accounted for; - retention, access control, and database
availability.

Those are application-level metering decisions, not license-verifier
decisions.

## 10.4 Failure behavior

For an enabled criterion, the following must deny startup: -
database/source unavailable; - required usage row missing; - value
cannot be parsed; - value is negative; - source returns malformed
data; - adapter raises an error; - current value is at or above the
signed limit.

Do not interpret source failure as zero usage. Treating a failed read as
zero would create a fail-open path.

## 10.5 Startup-only implications

The usage values are read at startup. If the application runs for
several hours and the host's token counter crosses its limit during that
process, the current design does not poll the database and does not
terminate the running process at that instant. The limit is enforced
when the next startup check occurs. A different requirement---such as
checking every operation---would require a separate design and could
materially affect latency, availability, and concurrency semantics.

------------------------------------------------------------------------

# 11. Developer configuration versus signed runtime policy

The project has a package-local developer configuration, including
identity, secret coordinates, criterion toggles, dates, limits, and
feature metadata. It is used for authoring/configuration and application
wiring. It is **not** the authoritative source for runtime limits once a
signed artifact has been retrieved.

A representative configuration from the implementation history was:

``` json
{
  "identity": {
    "license_id": "CLIENT-ALTERYX-001",
    "product": "alteryx-etl",
    "environment": "production"
  },
  "artifact_secret": {
    "secret_scope": "alteryx-licenseArtifacts",
    "secret_name": "alteryx-license"
  },
  "enforcement": {
    "date_enabled": 1,
    "volume_enabled": 0,
    "token_usage_enabled": 0
  },
  "date": {
    "issued_date": "2026-10-06T00:00:00Z",
    "expiry_date": "2026-11-06T00:00:00Z"
  },
  "volume": {
    "volume_limit": 9999
  },
  "token_usage": {
    "token_usage_limit": 500000
  },
  "features": {
    "workflow_analysis": true,
    "portfolio_rationalisation": true,
    "python_translation": true,
    "export_reports": true
  }
}
```

This is a historical example of the developer configuration shape, not a
guarantee that every value remains current. Check the deployed
`developer_config.json` before using it as a live configuration.

## 11.1 Authority rules

-   The signed artifact's policy determines which criteria are enforced.
-   The signed artifact's limits determine the Volume and Token Usage
    ceilings.
-   Local `date_enabled`, `volume_enabled`, `token_usage_enabled`, and
    local limits must not be OR-ed with or used as fallback values for
    the signed policy.
-   A local configuration change cannot authorize a modified artifact.
-   The policy must be mandatory in the artifact schema.
-   A signed policy with all criteria disabled must be rejected.

This separation prevents local configuration drift from silently
changing a license that was signed by the authorized issuer.

## 11.2 Environment variables and secrets

The actual license JSON should not be placed in `developer_config.json`,
committed to source control, or hardcoded in Python. In Databricks Apps,
the artifact is injected through `ALTERYX_LICENSE_ARTIFACT` using the
`valueFrom` resource mapping. Developer configuration can identify the
intended secret coordinates, but the App's runtime retrieval path must
use the supported App resource mechanism.

------------------------------------------------------------------------

# 12. Code organization and portability

The licensing package is organized as a self-contained module under:

``` text
backend/app/licensing/
├── __init__.py
├── config.py
├── developer_config.py
├── developer_config.json
├── errors.py
├── manager.py
├── models.py
├── secret_provider.py
├── usage_source.py
├── verifier.py
├── adapters/
│   ├── __init__.py
│   └── databricks.py
└── criteria/
    ├── __init__.py
    ├── base.py
    ├── date.py
    ├── volume.py
    └── token_usage.py
```

This tree reflects the implementation structure reported during
development. Confirm the actual current source tree before using it as a
strict packaging manifest.

## 12.1 Responsibility by module

  -----------------------------------------------------------------------
  Module                              Responsibility
  ----------------------------------- -----------------------------------
  `models.py`                         Typed models for the artifact and
                                      mandatory signed policy.

  `config.py`                         Local application/configuration
                                      values and validation.

  `developer_config.py` / `.json`     Developer-facing
                                      configuration/authoring inputs.

  `errors.py`                         Domain-specific licensing errors
                                      used for fail-closed handling.

  `manager.py`                        Orchestrates artifact retrieval,
                                      schema validation, signature
                                      verification, identity validation,
                                      and enabled criteria.

  `verifier.py`                       Cryptographic signature
                                      verification over canonical payload
                                      bytes.

  `secret_provider.py`                Generic secret-provider contract,
                                      without a hardcoded
                                      provider-specific special case.

  `usage_source.py`                   Generic usage-source contract.

  `criteria/base.py`                  Shared criterion
                                      interface/behavior.

  `criteria/date.py`                  Date validity evaluation.

  `criteria/volume.py`                Volume limit evaluation.

  `criteria/token_usage.py`           Token Usage limit evaluation.

  `adapters/databricks.py`            Databricks-specific integration at
                                      the boundary, including runtime
                                      secret injection/source support as
                                      implemented.
  -----------------------------------------------------------------------

The core must remain provider-agnostic. Databricks should be one
adapter, not an assumption embedded in the date/volume/token criterion
logic.

## 12.2 Explicit provider injection

`LicenseManager` should receive its secret provider explicitly. It
should not silently instantiate a Databricks provider inside the core.
This makes unit tests deterministic and permits another host to inject a
different provider.

Conceptual integration:

``` python
# Illustrative only; use the exact constructor signature in deployed code.
provider = DatabricksSecretProvider()
license_mgr = LicenseManager(secret_provider=provider)
license_mgr.validate_or_raise()
```

The actual application integration point in this project is
`backend/app/main.py`. The manager is invoked during the FastAPI
lifespan startup. Do not add separate licensing checks to every API
route or frontend component.

## 12.3 No frontend licensing

No frontend licensing implementation is required. The browser must not
receive the private signing key, and frontend state must not be the
authority for whether the application is licensed. The backend startup
gate is authoritative for the deployed service.

------------------------------------------------------------------------

# 13. Runtime startup sequence

The implemented application startup calls the licensing manager before
normal service initialization. The relevant operation is:

``` python
license_mgr.validate_or_raise()
```

The production sequence is:

1.  Databricks starts the App process.
2.  `app.yaml` maps the secret resource to `ALTERYX_LICENSE_ARTIFACT`.
3.  FastAPI begins its lifespan startup.
4.  The application injects the configured secret provider into
    `LicenseManager`.
5.  The manager retrieves the artifact text from the provider.
6.  JSON parsing and schema/model validation run.
7.  The signature is verified using the embedded public key.
8.  License identity is checked against the expected production
    identity.
9.  Each enabled criterion is evaluated.
10. If every check passes, normal initialization continues.
11. If any required check fails, the manager raises a licensing
    exception and startup fails.

The application should not catch a license-validation exception and
continue startup in an unlicensed state.

## 13.1 Startup failures

Fail-closed conditions include: - missing/empty artifact environment
variable; - invalid JSON; - missing mandatory `policy`; - missing
required fields or wrong field types; - missing or invalid Base64
signature; - signature mismatch; - incorrect license ID, product, or
environment; - expired date criterion; - volume source failure or volume
at/above limit when enabled; - token usage source failure or token usage
at/above limit when enabled; - invalid signed policy, including all
three criteria disabled.

A disabled usage criterion should not trigger a usage-source read solely
to enforce that disabled criterion. It must not be possible, however, to
disable a signed criterion using local configuration.

------------------------------------------------------------------------

# 14. Issuing a signed license in a controlled Databricks notebook

This section records the process used in the project. It is intended for
authorized license issuance only. Never run the signing flow from the
production App and never expose the private key.

The examples below focus on the cryptographic steps. The final payload
must be constructed using the exact required schema in the deployed
`backend/app/licensing/models.py`, including the mandatory `policy`
object. A legacy payload that lacks `policy` will fail schema
validation.

## 14.1 Prepare timestamps and identity

A historical date-only payload was prepared as follows:

``` python
from datetime import datetime, timedelta, timezone

issued_at = datetime.now(timezone.utc)
expires_at = issued_at + timedelta(days=30)

license_payload = {
    "license_id": "CLIENT-ALTERYX-001",
    "product": "alteryx-etl",
    "environment": "production",
    "issued_at": issued_at.isoformat(),
    "expires_at": expires_at.isoformat(),
    "features": {
        "workflow_analysis": True,
        "portfolio_rationalisation": True,
        "python_translation": True,
        "export_reports": True,
    },
}
```

**For the final policy-aware schema, add the mandatory `policy` object
with the exact structure expected by the deployed models before
signing.** Do not copy the legacy object above unchanged into production
and expect the final policy-aware verifier to accept it.

For example, the policy intent may be Date enabled, Volume disabled, and
Token Usage disabled, with limits defined as required by the schema. The
actual exact payload should be generated from the current
model/configuration code rather than manually approximated.

## 14.2 Retrieve the private key in the authorized signing environment

The historical notebook flow retrieved a Base64-encoded DER private key
from the signing scope:

``` python
import base64
from cryptography.hazmat.primitives.serialization import load_der_private_key

private_key_b64 = dbutils.secrets.get(
    scope="alteryx-licenseSigning",
    key="alteryx-license-private-key",
)

private_key_der = base64.b64decode(private_key_b64)

private_key = load_der_private_key(
    private_key_der,
    password=None,
)

print("Private key retrieved:", True)
print("Key type:", type(private_key).__name__)
```

Expected diagnostic output:

``` text
Private key retrieved: True
Key type: Ed25519PrivateKey
```

The secret is Base64-encoded DER rather than PEM text.
`base64.b64decode` restores the DER bytes, and `load_der_private_key`
parses those bytes into the cryptography key object.

Never print `private_key_b64`, `private_key_der`, or the private key
object. Notebook output should contain only safe status messages.

**Security boundary:** This is a signing/test scope flow. Do not attach
`alteryx-licenseSigning` to the production App. The production App must
not retrieve this private key.

## 14.3 Canonicalize and sign

``` python
import base64
import json

canonical_payload = json.dumps(
    license_payload,
    sort_keys=True,
    separators=(",", ":"),
    default=str,
).encode("utf-8")

signature = private_key.sign(canonical_payload)
signature_b64 = base64.b64encode(signature).decode("ascii")

signed_license = {
    **license_payload,
    "signature": signature_b64,
}

print("License signed:", True)
print("Signature length:", len(signature))
print("Signature Base64 length:", len(signature_b64))
```

For Ed25519, expected diagnostics are a 64-byte signature and an
88-character Base64 representation.

The actual `license_payload` must already include the mandatory signed
policy. Do not add the policy after this step; doing so would change the
signed bytes.

## 14.4 Verify the signature before publishing

``` python
import base64
import json
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey

embedded_public_key_b64 = (
    "aykIwjC0U0mxmTXUDhQdwBCiogj8YRNWy/8EieAfx9s="
)

public_key = Ed25519PublicKey.from_public_bytes(
    base64.b64decode(embedded_public_key_b64)
)

signature_to_verify = base64.b64decode(signed_license["signature"])

payload_to_verify = {
    key: value
    for key, value in signed_license.items()
    if key != "signature"
}

canonical_to_verify = json.dumps(
    payload_to_verify,
    sort_keys=True,
    separators=(",", ":"),
    default=str,
).encode("utf-8")

public_key.verify(signature_to_verify, canonical_to_verify)

print("Signature verification: PASSED")
print("Payload integrity: PASSED")
```

This verifies that the signature matches the payload and the embedded
public key. It does not replace the runtime's schema, identity, date,
Volume, or Token Usage checks. The verifier should use the same
canonicalization as the signer.

## 14.5 Serialize the final artifact

``` python
import json

license_json = json.dumps(
    signed_license,
    sort_keys=True,
    separators=(",", ":"),
)

with open("/tmp/alteryx-license.json", "w", encoding="utf-8") as handle:
    handle.write(license_json)

print("License artifact prepared:", True)
print("Artifact size:", len(license_json), "bytes")
print("Expires at:", license_payload["expires_at"])
```

Do not include a private key in this JSON. The `signature` field is a
public verification value, not the private key.

Before publishing, validate the complete artifact with the actual
deployed models/manager or an equivalent issuer-side validation path. Do
not publish an artifact merely because the low-level signature verifies;
schema and policy validation must also pass.

------------------------------------------------------------------------

# 15. Publishing and updating a license

The distribution path is the signed artifact vault, not the signing
vault.

## 15.1 Back up the currently valid artifact

A same-session Azure Cloud Shell backup can be made without echoing the
secret:

``` bash
VALID_LICENSE="$(
  az keyvault secret show \
    --vault-name alteryx-licenseArtifacts \
    --name alteryx-license \
    --query value \
    -o tsv
)"

if [ -n "$VALID_LICENSE" ]; then
  echo "Current license captured in this Cloud Shell session."
else
  echo "ERROR: Could not retrieve current license."
fi
```

Do not print `VALID_LICENSE`. This variable is only an immediate
rollback aid; use the approved secret-version or change-management
process for a durable recovery point.

## 15.2 Publish the signed JSON

Use the secure `read -s` and `az keyvault secret set` command in Section
5.4. Confirm that the command reports the expected secret name. Avoid
pasting the artifact into source code, `app.yaml`, or
`developer_config.json`.

## 15.3 What happens after the secret is updated?

The artifact in Key Vault changes, and the Databricks Key Vault-backed
scope is the distribution path used by the App resource. The app
consumes the injected environment value on process startup. Plan a
restart/redeployment as required by the platform's secret-resource
refresh behavior so the new value is present in the process environment.

Do not assume a running process dynamically refreshes its environment
variable. The design validates at startup and does not poll for a new
artifact.

## 15.4 Rollback

If the new artifact is invalid, restore the previously valid artifact
using the authorized rollback process, restart/redeploy the App as
needed, and verify successful license validation in the logs. After an
expiry kill-switch test, restore a valid artifact immediately;
otherwise, the app should remain blocked on startup.

------------------------------------------------------------------------

# 16. Volume and Token Usage: database setup and test bundle

A test-only SQLite bundle was used to verify the generic usage-source
contract and boundary behavior. It is not the production metering system
and must not be included in the protected production artifact.

The bundle contained: - `licensing_usage_test.db`; -
`seed_and_test.sql`; - `sqlite_usage_source_test.py`; - `README.md`.

The database was seeded with: - current volume = `100`; - current token
usage = `10000`.

The SQLite adapter is test-only. Production can use a different data
source through the generic `UsageDataSource` interface.

## 16.1 Boundary test cases

The reported boundary expectations were:

  -------------------------------------------------------------------------
  Criterion              Current value         Signed limit Expected
                                                            criterion
                                                            outcome
  --------------- -------------------- -------------------- ---------------
  Volume                           100                  500 Pass

  Volume                           500                  500 Fail

  Volume                           501                  500 Fail

  Token Usage                    10000               500000 Pass

  Token Usage                   500000               500000 Fail

  Token Usage                   500001               500000 Fail
  -------------------------------------------------------------------------

A test that expects the criterion to reject startup at the boundary
should itself report **PASS** when the expected licensing exception is
raised. Do not confuse "license decision denied" with "automated test
failed."

## 16.2 Additional reported tests

The test suite was reported to cover: - below-limit, exact-limit, and
above-limit cases for both criteria; - resetting the test values; -
signed limits taking precedence over local limits; - signed policy flags
taking precedence over local enabled flags; - missing database; -
missing usage row; - negative values; - malformed values; -
read-only/consumer-only semantics; - both usage criteria enabled
together; - direct database validation.

The test bundle is for validation only. Exclude
`licensing_usage_test_bundle`, test databases, seed files, and test-only
adapters from the production protected artifact.

## 16.3 How to use a different database

To port the package: 1. Define what Volume and Token Usage mean for the
target application. 2. Create or identify an authoritative store for the
two cumulative values. 3. Implement the project's `UsageDataSource`
contract for that store. 4. Ensure reads are deterministic and return
validated, non-negative values. 5. Raise an error on
unavailable/missing/malformed data; do not default to zero. 6. Inject
the adapter into the licensing manager/criteria according to the
deployed constructor contract. 7. Test below/exact/above boundaries and
failure cases. 8. Keep database credentials outside the signed artifact
and source code. 9. Ensure the production identity can read only the
required data. 10. Do not add counter writes or metering logic to the
licensing package unless that is a separately approved requirement.

------------------------------------------------------------------------

# 17. Test strategy and reported validation evidence

The licensing implementation was tested at several levels. Keep the raw
test output and deployment logs with the release record; summaries below
reproduce the reported results available during implementation.

## 17.1 Licensing tests

Earlier documentation recorded:

``` bash
pytest backend/tests/test_licensing.py -v
```

with `33 passed`. Later development expanded the licensing suite; a
subsequent reported result was **60 licensing tests passed**. Use the
current source tree and actual latest test output as the authoritative
release evidence.

The reported licensing coverage included: - valid signed artifact; -
invalid signature and wrong public key; - signed-field tampering; -
missing signature and invalid Base64; - malformed JSON and
missing/wrong-type fields; - mandatory policy validation; - license
identity checks; - expiry and expiry-boundary behavior; - UTC `Z`,
offset-aware, naive timestamp, and non-UTC offset cases; - missing/empty
secret and provider failures; - explicit provider injection; -
protection against local identity, secret-coordinate, and policy
overrides; - all criterion combinations and signed-policy authority.

## 17.2 Volume/Token Usage tests

The usage bundle test file was reported to contain **17 tests**.
Coverage included the boundaries and data-source failure modes described
in Section 16.

## 17.3 Full backend suite

A later report stated:

``` text
99 passed, 5 warnings
```

The same report's per-file test counts were not arithmetically
consistent with that total. For formal sign-off, preserve the actual
pytest output from the current release and use its summary line rather
than relying on a copied narrative count.

## 17.4 Compilation

The following command was reported successful:

``` bash
python -m compileall backend
```

## 17.5 Deployment validation

The valid-license Databricks App startup was reported successful after
correcting the App secret access path and issuing an artifact with the
mandatory `policy` field. The earlier missing-policy deployment failed
with a Pydantic schema-validation error; that failure demonstrated that
the deployed schema required the policy.

An expired signed artifact was also tested. The artifact's signature was
valid, but its expiry was in the past. The manager raised
`LicenseExpiredError`, and the application failed startup. This
demonstrates that cryptographic validity alone is not authorization: the
semantic policy checks must also pass.

The current deployment was later reported to be successfully running on
Databricks. Retain the actual App logs and release commit/version as the
definitive evidence.

------------------------------------------------------------------------

# 18. Kill-switch test procedure

Perform this only in an approved test window and with a clear recovery
path. Do not leave the production artifact expired.

1.  Back up the currently valid signed artifact using the approved
    process.
2.  Generate a test payload with the same expected identity and a past
    expiry.
3.  Include the mandatory signed policy in the exact current schema.
4.  Sign the complete canonical payload using the authorized private
    key.
5.  Verify the signature using the embedded public key.
6.  Publish the test artifact only to the intended test target---or, if
    testing production behavior is explicitly authorized, follow the
    approved production change process.
7.  Restart/redeploy the App so startup validation runs.
8.  Verify logs show `LicenseExpiredError` and startup failure.
9.  Restore the valid artifact.
10. Restart/redeploy again.
11. Verify successful license validation and normal application startup.

An example of an expired date is:

``` python
issued_at = datetime.now(timezone.utc)
expires_at = issued_at - timedelta(days=1)
```

This is only an illustration of the expiry test input. The complete
policy-aware payload still needs to conform to the current schema and be
signed after all fields are set.

**Never test the kill switch by manually editing `expires_at` in an
already signed artifact.** That tests signature rejection, not the
expiry criterion. The proper expiry test is a correctly signed artifact
whose expiry is in the past.

------------------------------------------------------------------------

# 19. Failure matrix and expected outcomes

  ------------------------------------------------------------------------
  Failure or condition     When relevant           Expected outcome
  ------------------------ ----------------------- -----------------------
  Artifact environment     Every startup           Fail closed
  variable missing/empty                           

  Malformed JSON           Every startup           Fail closed

  `policy` missing         Every startup           Schema validation fails

  Invalid signature        Every startup           Fail closed

  Signed field changed     Every startup           Signature verification
  after signing                                    fails

  Wrong license            Every startup           Identity validation
  ID/product/environment                           fails

  Date enabled and current Date enabled            Startup blocked
  time is at/after expiry                          

  Volume enabled and       Volume enabled          Criterion passes
  current volume is below                          
  limit                                            

  Volume enabled and       Volume enabled          Startup blocked
  current volume is                                
  at/above limit                                   

  Token Usage enabled and  Token Usage enabled     Criterion passes
  current token count is                           
  below limit                                      

  Token Usage enabled and  Token Usage enabled     Startup blocked
  current token count is                           
  at/above limit                                   

  Enabled usage source is  Corresponding criterion Fail closed
  unavailable              enabled                 

  Enabled usage value      Corresponding criterion Fail closed
  missing, malformed, or   enabled                 
  negative                                         

  All three policy flags   Policy authoring/schema Reject policy
  disabled                 validation              

  Local flag says disabled Runtime                 Signed policy wins
  but signed policy says                           
  enabled                                          

  Local limit is higher    Runtime                 Signed limit wins
  than signed limit                                

  Production App tries to  App runtime             Not a supported
  use `dbutils` for App                            retrieval path; use
  secret                                           injected environment
                                                   variable

  Production App has       Security                Remove access;
  access to signing scope  misconfiguration        production App must not
                                                   receive private key
  ------------------------------------------------------------------------

------------------------------------------------------------------------

# 20. Security model, threat coverage, and limitations

## 20.1 Controls provided

The design provides: - signed-artifact authenticity; - tamper detection
for signed payload and policy; - identity binding to the intended
license/product/environment; - date enforcement; - configurable Volume
and Token Usage thresholds; - fail-closed startup behavior; - separation
between license issuance and license verification; - separation of
generic core logic from platform-specific adapters.

## 20.2 Threats addressed

  -----------------------------------------------------------------------
  Threat                              Expected control
  ----------------------------------- -----------------------------------
  Extend expiry by editing JSON       Signature verification fails

  Increase volume limit by editing    Signature verification fails
  JSON                                

  Increase token limit by editing     Signature verification fails
  JSON                                

  Disable a criterion by editing JSON Signature verification fails

  Change license                      Signature and/or identity
  ID/product/environment              validation fails

  Remove signature                    Schema/verification fails

  Corrupt JSON                        Parsing/schema validation fails

  Remove artifact secret              Retrieval fails closed

  Exceed enabled usage limit          Startup blocked

  Make usage source unavailable       Startup blocked for an enabled
                                      usage criterion

  Generate a new license without      Cannot produce a valid signature
  signing key                         under the embedded public key
  -----------------------------------------------------------------------

## 20.3 What the design does not claim

No software-only licensing system can guarantee that a sufficiently
privileged attacker who can alter the runtime, replace the public key,
patch the verifier, or modify the deployed application can never bypass
it. The design protects the issuance authority and provides verifiable
authenticity/integrity for the signed policy; it is not a mathematical
guarantee against total runtime compromise.

The usage criteria also depend on the integrity of the usage source. A
signed limit is only as meaningful as the current counter being read.
Access control, write authority, auditing, and consistency of the host's
usage database remain important.

The startup-only model does not enforce limits immediately in a running
process. If immediate enforcement or per-operation quota reservation is
required, that is a different system design.

------------------------------------------------------------------------

# 21. Porting the licensing package to another application

The package is designed to be embedded in another Python application
without making Databricks a core dependency.

## 21.1 Components to keep

Keep the generic components: - artifact/policy models; -
canonicalization and signature verifier; - license manager; - error
types; - Date, Volume, and Token Usage criteria; - generic
secret-provider contract; - generic usage-source contract.

## 21.2 Components to replace or configure

Supply adapters for: - retrieving the signed artifact from the target
platform's secret manager; - reading usage counters from the target
application's database; - application startup integration; - expected
license identity and authoring configuration.

Examples of possible secret providers or usage stores depend on the host
environment; they are not all implemented in this project. Do not import
the Databricks adapter into a non-Databricks deployment unless that
environment actually requires it.

## 21.3 Integration checklist

1.  Copy the licensing package into the target application's package
    structure.
2.  Preserve the signed artifact models and mandatory policy validation.
3.  Configure the correct public key for the license issuer.
4.  Keep the private key only in the authorized signing environment.
5.  Implement and inject the target secret provider.
6.  Implement and inject the usage data source if Volume or Token Usage
    will be enabled.
7.  Define authoritative semantics for each usage counter.
8.  Configure the expected license identity.
9.  Invoke `validate_or_raise()` once during application startup.
10. Ensure startup exceptions prevent the app from accepting traffic.
11. Keep signed policy as the sole runtime enforcement authority.
12. Add tests for all eight policy combinations, signature tampering,
    expiry boundaries, usage boundaries, and data-source failures.
13. Exclude signing keys, live secrets, `.env` files, test databases,
    and test-only adapters from production packages.
14. Validate the final artifact using the actual deployed models before
    publishing.
15. Record the test output, release version, and artifact version for
    auditability.

## 21.4 Portability principle

The reusable core should depend on **interfaces**, not a cloud provider.
A provider-specific adapter may use platform facilities, but the
Date/Volume/Token Usage criteria should not know whether the app runs on
Databricks, a container platform, a VM, or another Python host.

------------------------------------------------------------------------

# 22. Databricks deployment and protected packaging considerations

The licensing implementation was validated first in source form on
Databricks. That sequence is important: confirm secret injection, schema
validity, usage-source behavior, startup gating, and application health
before compiling or sealing the package.

The project also investigated Nuitka package-mode protection for the AWA
and backend application modules. The protected deployment target was
designed to avoid distributing proprietary `.py` implementation files
while retaining required runtime resources.

## 22.1 Production artifact exclusions

Do not include: - Ed25519 private signing key; - LLM API keys or other
live credentials; - `.env` files; - test databases and usage-test
bundles; - test-only adapters; - unnecessary tests and development
artifacts; - proprietary `.py` implementation files when the protected
build requirement is to compile them; - `.pyc` and `__pycache__`
artifacts.

Keep required non-code runtime resources, such as the parser grammar
file, where the compiled package expects them. Do not remove resources
merely because they are not Python source.

## 22.2 Nuitka package-mode commands used during development

The following are historical commands used to compile Python packages in
the Linux/Python 3.11 build environment. Run them only in the matching
build environment and verify the current source paths first.

Compile AWA package (historical top-level package command):

``` bash
PYTHONPATH=backend \
/tmp/nuitka311/bin/python -m nuitka \
  --mode=package \
  --include-package=awa \
  backend/awa \
  --output-dir=deployment/awa_package_py311 \
  --remove-output
```

The project later confirmed that the canonical source namespace was
`backend.awa`, not top-level `awa`, and compiled it with:

``` bash
PYTHONPATH=. \
/tmp/nuitka311/bin/python -m nuitka \
  --mode=package \
  --include-package=backend.awa \
  backend/awa \
  --output-dir=deployment/backend_awa_package_py311 \
  --remove-output
```

Compile the backend application package:

``` bash
PYTHONPATH=. \
/tmp/nuitka311/bin/python -m nuitka \
  --mode=package \
  --include-package=backend.app \
  backend/app \
  --output-dir=deployment/app_package_py311 \
  --remove-output
```

These commands are not required to use the licensing package in source
form. They are included as part of the wider packaging history and
should be revalidated against the current code, Python ABI, OS
architecture, and Databricks App constraints before reuse.

## 22.3 Licensing must behave identically after compilation

Compilation must not change: - the signed policy schema; -
canonicalization; - public key used by the verifier; - the secret
provider's runtime behavior; - usage source semantics; - startup-only
enforcement; - fail-closed behavior.

The protected artifact must not include the private key. The production
App must still receive the signed artifact through the configured
Databricks App secret resource.

## 22.4 Source-first deployment validation

Before creating a new protected build, verify in source deployment: -
app startup with a valid signed artifact; - startup rejection for an
expired but correctly signed artifact; - rejection of a malformed or
tampered artifact; - rejection when an enabled usage criterion reaches
its limit; - rejection when an enabled usage source is unavailable; -
success when each enabled criterion is below its limit; - `/api/health`
and core application routes; - frontend root and static asset serving if
the app includes the React frontend.

Only after these checks pass should the same behavior be verified in the
compiled deployment.

------------------------------------------------------------------------

# 23. Operational runbooks

## 23.1 Issue or renew a license

1.  Confirm the intended license ID, product, environment, issue time,
    expiry time, features, enabled criteria, and limits.
2.  Construct the complete payload using the current schema, including
    mandatory `policy`.
3.  Retrieve the signing key only in the authorized signing environment.
4.  Canonicalize and sign the complete payload.
5.  Verify the signature with the embedded public key.
6.  Validate the artifact against the current model/schema and policy
    rules.
7.  Back up the current valid artifact using the approved change
    process.
8.  Publish the new signed JSON to `alteryx-licenseArtifacts` /
    `alteryx-license`.
9.  Ensure the App resource and `app.yaml` mapping remain correct.
10. Restart/redeploy as required to refresh the injected secret.
11. Confirm the App logs show successful validation and startup.
12. Record the artifact version, expiry, policy limits, deployment ID,
    and test evidence.

## 23.2 Raise or lower a usage limit

1.  Change the intended limit in the authoring configuration.
2.  Build the full signed payload with the updated policy.
3.  Sign the entire payload with the authorized private key.
4.  Verify and schema-validate the new artifact.
5.  Publish it through the artifact vault.
6.  Restart/redeploy as required.
7.  Confirm the runtime is using the new artifact and the expected
    limit.
8.  Do not edit the JSON in Key Vault manually to change a limit; doing
    so invalidates the signature.

## 23.3 Disable or enable a criterion

1.  Update the intended policy flag in the license authoring flow.
2.  Confirm at least one of Date, Volume, or Token Usage remains
    enabled.
3.  Sign the complete payload.
4.  Verify signature and schema.
5.  Publish and restart/redeploy.
6.  Confirm startup logs and test behavior.

Changing a local developer flag alone does not change the signed runtime
policy.

## 23.4 Respond to startup failure

1.  Inspect the specific licensing exception; do not assume all failures
    mean expiry.
2.  Determine whether the failure is secret retrieval, JSON/schema
    validation, signature, identity, date, usage source, or usage
    threshold.
3.  Do not disable the kill switch as a quick fix.
4.  If artifact schema is invalid, regenerate it from the current model.
5.  If signature verification fails, regenerate and sign the artifact;
    do not edit the signature or payload manually.
6.  If an enabled usage source is unavailable, restore the
    source/permissions or deliberately issue a newly authorized policy.
7.  If the artifact was replaced incorrectly, restore the previous
    known-good signed artifact and restart/redeploy.
8.  Record the incident and the artifact/deployment versions.

------------------------------------------------------------------------

# 24. Configuration and release checklist

## Azure and Databricks resources

-   [ ] `alteryx-licensing` contains `alteryx-license-private-key`.
-   [ ] The private-key vault is restricted to authorized signing
    identities.
-   [ ] `alteryx-licenseArtifacts` contains `alteryx-license`.
-   [ ] Databricks scope `alteryx-licenseArtifacts` is backed by the
    intended artifact vault.
-   [ ] Production App Secret resource references scope
    `alteryx-licenseArtifacts` and key `alteryx-license`.
-   [ ] Resource key is `license-artifact`.
-   [ ] Resource permission is `Can read`.
-   [ ] `app.yaml` maps `ALTERYX_LICENSE_ARTIFACT` from
    `license-artifact`.
-   [ ] Production App has no access to `alteryx-licenseSigning`.
-   [ ] LLM credentials remain separate from licensing credentials.

## Signed artifact

-   [ ] JSON parses.
-   [ ] Required fields exist.
-   [ ] Mandatory `policy` is present.
-   [ ] Policy enables at least one criterion.
-   [ ] Enabled limits and date values are valid.
-   [ ] Identity is correct.
-   [ ] Signature verifies using the embedded public key.
-   [ ] Artifact has not been manually edited after signing.
-   [ ] New artifact is published to the artifact vault, not the signing
    vault.

## Usage source

-   [ ] Usage semantics are documented.
-   [ ] Current volume and token values are authoritative.
-   [ ] Reads are read-only.
-   [ ] Missing/negative/malformed values fail closed.
-   [ ] Below/exact/above limit tests pass.
-   [ ] Test database and test adapter are excluded from production.

## Application

-   [ ] `LicenseManager` receives the provider explicitly.
-   [ ] App runtime reads the injected environment variable, not
    notebook-only `dbutils`.
-   [ ] Validation runs during startup.
-   [ ] Validation errors prevent startup.
-   [ ] No periodic polling or request-time checks are assumed.
-   [ ] Valid-license startup passes.
-   [ ] Expired-license startup fails.
-   [ ] Enabled usage threshold tests behave as expected.
-   [ ] Existing application/API health checks pass.
-   [ ] Protected build, if used, has been tested separately from source
    deployment.

------------------------------------------------------------------------

# 25. Key commands reference

## Validate Python syntax

``` bash
python -m compileall backend
```

## Run licensing tests

``` bash
pytest backend/tests/test_licensing.py -v
```

## Run usage-bundle tests

``` bash
pytest backend/tests/test_licensing_usage_bundle.py -v
```

## Run all backend tests

``` bash
pytest backend/tests -v
```

## Inspect artifact-secret metadata only

``` bash
az keyvault secret show \
  --vault-name alteryx-licenseArtifacts \
  --name alteryx-license \
  --query "{name:name, enabled:attributes.enabled}" \
  -o json
```

## Publish a signed artifact without echoing it

``` bash
read -s -p "Paste signed license JSON: " LICENSE_JSON
echo

az keyvault secret set \
  --vault-name alteryx-licenseArtifacts \
  --name alteryx-license \
  --value "$LICENSE_JSON" \
  --query "name" \
  -o tsv

unset LICENSE_JSON
```

The publishing commands do not generate or sign a license. They only
publish a JSON artifact that must already have been generated, signed,
and validated.

------------------------------------------------------------------------

# 26. Implementation history and important corrections

The following corrections are particularly important because they were
encountered during real deployment.

## 26.1 Runtime `dbutils` assumption

An initial production App deployment attempted to resolve `dbutils` from
the App process. Startup failed because `dbutils` was unavailable in
that environment. The App's Secret resource and `app.yaml` `valueFrom`
mapping are the correct runtime delivery mechanism; application code
reads `ALTERYX_LICENSE_ARTIFACT`.

## 26.2 Mandatory policy schema

After the policy-aware model was introduced, an artifact generated with
only the earlier identity/date/features fields failed with a Pydantic
error indicating that `policy` was required. The fix was to generate and
sign an artifact containing the mandatory policy, not to remove the
requirement from the model.

## 26.3 Signed policy is authoritative

A correction removed the runtime behavior that could merge local
enforcement flags with artifact policy. The signed artifact is the sole
runtime authority for which criteria are enabled and what the usage
limits are. The local developer configuration is not a bypass.

## 26.4 Provider injection

The manager was corrected to use explicit secret-provider injection
rather than silently constructing a Databricks provider inside the core.
This is essential to portability and deterministic testing.

## 26.5 Usage test outcomes

At exact/above limit, the **license criterion should reject startup**.
An automated test passes when it observes that expected rejection. The
test result and license decision are different concepts.

## 26.6 Frontend routing is separate from licensing

The React frontend is served by the FastAPI application, while licensing
gates backend startup. A root route returning HTML or static assets
returning 404 is a frontend-serving issue, not evidence that the license
was bypassed or failed. Keep frontend routing changes separate from the
licensing policy and secret integration.

------------------------------------------------------------------------

# 27. Final architecture summary

``` text
                   AUTHORIZED LICENSE ISSUER
                              |
                              | reads private key
                              v
             Azure Key Vault: alteryx-licensing
             Secret: alteryx-license-private-key
                              |
                              v
                Canonicalize complete payload
                 (identity + dates + features
                    + mandatory signed policy)
                              |
                              v
                        Ed25519 sign
                              |
                              v
                    Verify before publishing
                              |
                              v
          Azure Key Vault: alteryx-licenseArtifacts
                  Secret: alteryx-license
                              |
                              v
         Databricks KV-backed secret scope:
                   alteryx-licenseArtifacts
                              |
                              v
             Databricks App Secret resource
                   resource key: license-artifact
                              |
                              v
             app.yaml -> ALTERYX_LICENSE_ARTIFACT
                              |
                              v
                     Databricks App startup
                              |
                              v
                         LicenseManager
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
             Schema       Signature      Identity
             + policy     verification   validation
                |
                v
         Evaluate enabled signed criteria
                |
       +--------+------------------+
       |              |             |
       v              v             v
      Date          Volume      Token Usage
       |              |             |
       |        read-only usage source
       |              |             |
       +--------------+-------------+
                      |
                      v
        All enabled criteria pass?
                 /          \
               YES           NO
                |             |
                v             v
          App startup     Fail closed
           continues    startup blocked
```

## Final operational facts

1.  `alteryx-licensing` stores the private signing key; the production
    App must not access it.
2.  `alteryx-licenseArtifacts` stores the signed JSON artifact consumed
    by the production App.
3.  There is no direct vault-to-vault synchronization in the application
    flow; an authorized signer publishes the signed artifact to the
    artifact vault.
4.  Databricks delivers the artifact through a secret scope and App
    Secret resource into `ALTERYX_LICENSE_ARTIFACT`.
5.  The production App uses the public key to verify the signature; it
    never needs the private key.
6.  The signed policy is mandatory and is the sole runtime authority for
    enabled criteria and limits.
7.  Date, Volume, and Token Usage are independently configurable, but
    all enabled criteria must pass.
8.  Volume and Token Usage read cumulative values from a configured
    source; the licensing package does not meter, increment, or write
    usage.
9.  Source/data errors for enabled criteria fail closed.
10. The checks are startup-only. They do not instantly terminate a
    running process at expiry or when a usage counter later crosses a
    limit.
11. The package is portable through explicit secret-provider and
    usage-source adapters.
12. Never distribute private keys, live credentials, `.env` files, or
    test usage databases with the production artifact.

------------------------------------------------------------------------

**End of guide.**
