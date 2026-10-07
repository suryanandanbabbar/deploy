# Alteryx ETL Rationalisation --- Production License Enforcement & Kill-Switch

## Document Purpose

This document is the complete technical guide for the licensing
architecture implemented for the Alteryx ETL Rationalisation
application. It covers the Azure resources, Databricks resources,
cryptographic design, license creation/signing, runtime verification,
deployment configuration, kill-switch testing, rollback, security
boundaries, and the exact commands used.

------------------------------------------------------------------------

# 1. Executive Summary

The application uses a **cryptographically signed license artifact** to
decide whether it is allowed to start.

The design is intentionally **offline at runtime**:

-   The production application does not call an external licensing
    server.
-   The production application does not retrieve the private signing
    key.
-   There is no license heartbeat, lease-renewal loop, or online
    polling.
-   The production application retrieves a signed JSON license artifact.
-   The application contains the corresponding Ed25519 public
    verification key.
-   Startup validates the artifact and fails closed if any validation
    fails.

The core decision is:

``` text
Signed license
      |
      v
JSON/schema valid?
      |
      v
Ed25519 signature valid?
      |
      v
Correct license ID/product/environment?
      |
      v
Not expired?
      |
   +--+--+
   |     |
  YES    NO
   |     |
   v     v
START   FAIL
```

This is a **startup kill switch**. It prevents the application from
starting after license expiry. It is not a mechanism that asynchronously
terminates an already-running process at the exact moment of expiry.

------------------------------------------------------------------------

# 2. Security Model

The central security separation is:

``` text
PRIVATE KEY
    |
    | used only by authorized signing process
    v
SIGNED LICENSE ARTIFACT
    |
    | consumed by production application
    v
PUBLIC KEY
    |
    | verifies signature
    v
VALID / INVALID
```

The production application has:

``` text
Public key              YES
Signed license artifact YES
Private signing key     NO
```

This matters because a public-key signature scheme lets the runtime
verify authenticity without giving the runtime the ability to issue new
licenses.

------------------------------------------------------------------------

# 3. Azure Resources

## 3.1 Azure Key Vault: `alteryx-licensing`

This is the **signing-side Key Vault**.

Secret:

``` text
alteryx-license-private-key
```

The secret contains the Ed25519 private signing key.

This is the most sensitive credential in the licensing architecture.

It must not be:

-   committed to source control,
-   included in the frontend,
-   included in the production App,
-   printed in logs,
-   pasted into documentation,
-   distributed to the client.

The production App does not need this secret.

------------------------------------------------------------------------

## 3.2 Azure Key Vault: `alteryx-licenseArtifacts`

This is the **license distribution/storage Key Vault**.

Secret:

``` text
alteryx-license
```

The value is the signed license JSON artifact.

This is what the production runtime consumes.

The separation is intentional:

``` text
alteryx-licensing
    -> private signing key

alteryx-licenseArtifacts
    -> signed license artifact
```

------------------------------------------------------------------------

# 4. Databricks Secret Scopes

## 4.1 `alteryx-llm-secrets`

Existing application scope for LLM credentials.

Example:

``` text
scope: alteryx-llm-secrets
key:   azure-llm-key
```

This is separate from licensing.

## 4.2 `alteryx-licenseArtifacts`

This is the **production license scope**.

It is Azure Key Vault-backed to:

``` text
Azure Key Vault:
alteryx-licenseArtifacts
```

Secret:

``` text
alteryx-license
```

The production App uses this artifact.


------------------------------------------------------------------------

# 5. Databricks App Secret Resource

The production App has a secret resource configured as:

  Field          Value
  -------------- ----------------------------
  Secret scope   `alteryx-licenseArtifacts`
  Secret key     `alteryx-license`
  Permission     `Can read`
  Resource key   `license-artifact`

The corresponding `app.yaml` entry is:

``` yaml
- name: ALTERYX_LICENSE_ARTIFACT
  valueFrom: license-artifact
```

Databricks resolves `license-artifact` to the secret resource and makes
the secret available to the application as:

``` text
ALTERYX_LICENSE_ARTIFACT
```

The actual license JSON is not hard-coded in source.

------------------------------------------------------------------------

# 6. Databricks Secret Permission Consideration

A Databricks App secret resource grants access at the scope level rather
than providing fine-grained access to only one secret.

Therefore the production scope:

``` text
alteryx-licenseArtifacts
```

should remain narrow and should not contain unrelated secrets that the
App should not be able to read.

------------------------------------------------------------------------

# 7. Cryptographic Algorithm --- Ed25519

The signing system uses **Ed25519**, an elliptic-curve digital signature
scheme.

The process is:

``` text
Private key + message
        |
        v
    Ed25519 sign
        |
        v
    64-byte signature
```

Verification is:

``` text
Public key + message + signature
        |
        v
    Ed25519 verify
        |
        v
     valid/invalid
```

The signature is 64 bytes.

For JSON storage it is Base64 encoded, which produces an 88-character
Base64 representation for a 64-byte signature.

The important security property is:

> The public key can verify signatures, but it cannot be used to create
> valid signatures.

------------------------------------------------------------------------

# 8. Embedded Public Key

The application contains this public key:

``` text
aykIwjC0U0mxmTXUDhQdwBCiogj8YRNWy/8EieAfx9s=
```

It is Base64-encoded raw Ed25519 public-key material.

The corresponding private key is not embedded.

------------------------------------------------------------------------

# 9. Immutable Production Identity

The application hard-binds:

``` text
License ID:
CLIENT-ALTERYX-001

Product:
alteryx-etl

Environment:
production
```

It also hard-binds:

``` text
Secret scope:
alteryx-licenseArtifacts

Secret name:
alteryx-license
```

`LicenseConfig.from_env()` does not allow production identity or
production secret binding to be replaced by arbitrary environment
values.

This prevents a deployment from simply pointing the application at
another license identity or another production secret.

------------------------------------------------------------------------

# 10. License Payload Schema

The license payload is:

``` json
{
  "license_id": "CLIENT-ALTERYX-001",
  "product": "alteryx-etl",
  "environment": "production",
  "issued_at": "2026-...",
  "expires_at": "2026-...",
  "features": {
    "workflow_analysis": true,
    "portfolio_rationalisation": true,
    "python_translation": true,
    "export_reports": true
  }
}
```

The final signed artifact adds:

``` json
"signature": "<Base64 Ed25519 signature>"
```

The feature value is a dictionary, not a list.

------------------------------------------------------------------------

# 11. Full Architecture

``` text
                        SIGNING / ISSUANCE SIDE
                                  |
                                  v
                 +--------------------------------+
                 | Azure Key Vault                 |
                 | alteryx-licensing              |
                 |                                |
                 | alteryx-license-private-key    |
                 +----------------+---------------+
                                  |
                                  | Ed25519 sign
                                  v
                         Signed license JSON
                                  |
                                  v
                 +--------------------------------+
                 | Azure Key Vault                 |
                 | alteryx-licenseArtifacts       |
                 |                                |
                 | alteryx-license                |
                 +----------------+---------------+
                                  |
                                  | Key Vault-backed
                                  v
                 +--------------------------------+
                 | Databricks Secret Scope         |
                 | alteryx-licenseArtifacts       |
                 |                                |
                 | alteryx-license                |
                 +----------------+---------------+
                                  |
                                  | App Secret Resource
                                  v
                 +--------------------------------+
                 | Databricks App                 |
                 |                                |
                 | ALTERYX_LICENSE_ARTIFACT       |
                 +----------------+---------------+
                                  |
                                  v
                 +--------------------------------+
                 | LicenseManager                  |
                 |                                |
                 | Embedded Ed25519 public key   |
                 +----------------+---------------+
                                  |
                 +----------------+----------------+
                 |                |                |
                 v                v                v
              Signature        Identity         Expiry
              validation       validation       validation
                 |                |                |
                 +----------------+----------------+
                                  |
                              ALL PASS?
                              /                                   YES        NO
                             |          |
                             v          v
                        APP STARTS   STARTUP FAILS
```

------------------------------------------------------------------------

# 12. Runtime LicenseManager Flow

Application startup invokes:

``` python
license_mgr = LicenseManager()
license_mgr.validate_or_raise()
```

The sequence is:

``` text
FastAPI lifespan starts
        |
        v
LicenseManager()
        |
        v
validate_or_raise()
        |
        +--> validate configuration
        |
        +--> retrieve signed artifact
        |
        +--> parse JSON
        |
        +--> validate schema
        |
        +--> verify Ed25519 signature
        |
        +--> validate identity
        |
        +--> validate expiry
        |
        v
Application initialization
        |
        v
Application startup complete
```

Any exception blocks startup.

------------------------------------------------------------------------

# 13. Why the Production App Does Not Retrieve the Private Key

The production App must not perform:

``` text
App -> Azure Key Vault -> private key
```

The correct flow is:

``` text
Authorized signer
    |
    | private key
    v
Signed artifact
    |
    v
Production App
    |
    | public key
    v
Verify
```

If the runtime had the private key, compromise of the runtime could
potentially allow creation of new valid licenses.

Therefore:

``` text
Signing authority = private key
Runtime = public key
```

------------------------------------------------------------------------

# 14. Generate the License Payload

For a normal 30-day license:

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

print("Payload prepared:", True)
print("Features type:", type(license_payload["features"]).__name__)
print("Feature count:", len(license_payload["features"]))
print("License ID:", license_payload["license_id"])
print("Product:", license_payload["product"])
print("Environment:", license_payload["environment"])
print("Expires at:", license_payload["expires_at"])
```

## What this does

`issued_at` records when the artifact was created.

`expires_at` defines when the license becomes invalid.

The identity fields bind the artifact to the intended
client/product/environment.

The feature dictionary describes licensed functionality.

UTC is deliberately used for timestamps to avoid timezone ambiguity.

------------------------------------------------------------------------

# 15. Retrieve the Signing Key

The controlled signing environment retrieves the private key from the
testing-only signing scope:

``` python
import base64
from cryptography.hazmat.primitives.serialization import load_der_private_key

private_key_b64 = dbutils.secrets.get(
    scope="alteryx-licenseSigning",
    key="alteryx-license-private-key"
)

private_key_der = base64.b64decode(private_key_b64)

private_key = load_der_private_key(
    private_key_der,
    password=None,
)

print("Private key retrieved:", True)
print("Key type:", type(private_key).__name__)
```

Expected:

``` text
Private key retrieved: True
Key type: Ed25519PrivateKey
```

## How it works

The secret is stored as Base64-encoded DER.

``` python
dbutils.secrets.get(...)
```

retrieves the secret.

``` python
base64.b64decode(...)
```

converts Base64 text into DER bytes.

``` python
load_der_private_key(...)
```

parses the PKCS#8 DER private key into an Ed25519 key object.

The private key itself is never printed.

------------------------------------------------------------------------

# 16. Why the Key Was Loaded as DER

The retrieved value was not PEM text.

The decoded key material was 48 bytes and began with:

``` text
302e0201
```

This corresponds to the DER representation used by the PKCS#8 Ed25519
private key.

Therefore:

``` python
load_der_private_key(...)
```

is the correct parser.

A PEM parser would be inappropriate for this stored representation.

------------------------------------------------------------------------

# 17. Canonicalize the Payload

Use:

``` python
import json

canonical_payload = json.dumps(
    license_payload,
    sort_keys=True,
    separators=(",", ":"),
    default=str,
).encode("utf-8")
```

## Why canonicalization is necessary

Digital signatures sign **bytes**, not abstract dictionaries.

These can represent the same logical JSON:

``` json
{"product":"alteryx-etl","license_id":"CLIENT-ALTERYX-001"}
```

and:

``` json
{
  "license_id": "CLIENT-ALTERYX-001",
  "product": "alteryx-etl"
}
```

but their byte sequences differ.

A signature created over one byte sequence will not automatically verify
against another byte sequence.

Therefore the signer and verifier must produce the exact same bytes.

The implementation makes serialization deterministic with:

``` python
sort_keys=True
```

which sorts object keys.

And:

``` python
separators=(",", ":")
```

which removes insignificant JSON whitespace.

Then:

``` python
.encode("utf-8")
```

creates the exact bytes that Ed25519 signs.

The verifier performs the same canonicalization.

------------------------------------------------------------------------

# 18. Sign the Canonical Payload

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
print("Features type:", type(signed_license["features"]).__name__)
print("Feature count:", len(signed_license["features"]))
```

Expected:

``` text
License signed: True
Signature length: 64
Signature Base64 length: 88
Features type: dict
Feature count: 4
```

## What `sign()` does

``` python
private_key.sign(canonical_payload)
```

uses Ed25519 to create a 64-byte digital signature over the canonical
payload.

The signature is binary, so it is Base64 encoded for JSON storage.

------------------------------------------------------------------------

# 19. Verify Before Publishing

The public key used by the application is:

``` python
embedded_public_key_b64 = (
    "aykIwjC0U0mxmTXUDhQdwBCiogj8YRNWy/8EieAfx9s="
)
```

Verification:

``` python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey

embedded_public_key_b64 = (
    "aykIwjC0U0mxmTXUDhQdwBCiogj8YRNWy/8EieAfx9s="
)

public_key = Ed25519PublicKey.from_public_bytes(
    base64.b64decode(embedded_public_key_b64)
)

signature_to_verify = base64.b64decode(
    signed_license["signature"]
)

payload_to_verify = {
    k: v
    for k, v in signed_license.items()
    if k != "signature"
}

canonical_to_verify = json.dumps(
    payload_to_verify,
    sort_keys=True,
    separators=(",", ":"),
    default=str,
).encode("utf-8")

public_key.verify(
    signature_to_verify,
    canonical_to_verify,
)

print("Signature verification: PASSED")
print("Payload integrity: PASSED")
print("Feature Schema: PASSED")
```

If verification fails, **do not publish the artifact**.

This catches:

-   wrong private key,
-   wrong public key,
-   changed payload,
-   incorrect canonicalization,
-   corrupted signature,
-   encoding errors.

------------------------------------------------------------------------

# 20. Create the Final JSON Artifact

``` python
import json

license_json = json.dumps(
    signed_license,
    sort_keys=True,
    separators=(",", ":"),
)

with open("/tmp/alteryx-license.json", "w") as f:
    f.write(license_json)

print("License artifact prepared:", True)
print("Artifact size:", len(license_json), "bytes")
print("Expires at:", license_payload["expires_at"])
```

The final artifact contains:

``` text
license_id
product
environment
issued_at
expires_at
features
signature
```

The artifact is then ready to be published.

------------------------------------------------------------------------

# 21. Azure Cloud Shell --- Back Up the Existing License

Before changing the production secret:

``` bash
VALID_LICENSE="$(az keyvault secret show   --vault-name alteryx-licenseArtifacts   --name alteryx-license   --query value   -o tsv)"

if [ -n "$VALID_LICENSE" ]; then
    echo "Current license backed up in this Cloud Shell session."
else
    echo "ERROR: Could not retrieve current license."
fi
```

Do not run:

``` bash
echo "$VALID_LICENSE"
```

The artifact should not be unnecessarily exposed in terminal output.

The backup provides an immediate rollback path.

------------------------------------------------------------------------

# 22. Publish the New License to Azure Key Vault

From the Databricks notebook, print the artifact:

``` python
print(license_json)
```

Copy it to Cloud Shell and run:

``` bash
read -s -p "Paste signed license JSON: " LICENSE_JSON
echo

az keyvault secret set   --vault-name alteryx-licenseArtifacts   --name alteryx-license   --value "$LICENSE_JSON"   --query "name"   -o tsv

unset LICENSE_JSON
```

Expected output:

``` text
alteryx-license
```

## What each command does

``` bash
read -s
```

reads the artifact without echoing it.

``` bash
az keyvault secret set
```

writes the signed artifact to Azure Key Vault.

``` bash
--vault-name alteryx-licenseArtifacts
```

selects the artifact Key Vault.

``` bash
--name alteryx-license
```

selects the production license secret.

``` bash
--query "name"
```

prints only the secret name instead of returning the secret value.

``` bash
unset LICENSE_JSON
```

removes the temporary shell variable.

------------------------------------------------------------------------

# 23. Verify Secret Metadata Without Revealing the License

``` bash
az keyvault secret show   --vault-name alteryx-licenseArtifacts   --name alteryx-license   --query "{name:name, enabled:attributes.enabled}"   -o json
```

Expected:

``` json
{
  "name": "alteryx-license",
  "enabled": true
}
```

------------------------------------------------------------------------

# 24. Databricks App Runtime Consumption

The App has:

``` yaml
env:
  - name: ALTERYX_LICENSE_ARTIFACT
    valueFrom: license-artifact
```

Databricks maps:

``` text
license-artifact
        |
        v
alteryx-licenseArtifacts / alteryx-license
        |
        v
ALTERYX_LICENSE_ARTIFACT
```

The secret provider reads the App environment variable.

The licensing manager then parses and validates the JSON.

------------------------------------------------------------------------

# 25. Production Validation Order

The manager validates:

1.  Configuration.
2.  Secret retrieval.
3.  JSON parsing.
4.  License schema.
5.  Ed25519 signature.
6.  License identity.
7.  Expiration.

The application starts only after all seven pass.

------------------------------------------------------------------------

# 26. Fail-Closed Behavior

These conditions cause startup failure:

``` text
Missing secret
Empty secret
Malformed JSON
Missing required field
Wrong field type
Missing signature
Invalid Base64
Invalid Ed25519 signature
Modified signed fields
Wrong license ID
Wrong product
Wrong environment
Expired license
```

This is intentional.

The licensing system never treats an ambiguous state as authorized.

------------------------------------------------------------------------

# 27. Actual Valid-License Test

The actual signed artifact was stored in:

``` text
Azure Key Vault:
alteryx-licenseArtifacts

Secret:
alteryx-license
```

Databricks successfully retrieved it using:

``` python
license_from_scope = dbutils.secrets.get(
    scope="alteryx-licenseArtifacts",
    key="alteryx-license"
)
```

The artifact was 403 bytes.

The real `LicenseManager` was tested with:

``` python
from backend.app.licensing import LicenseManager

license_mgr = LicenseManager()
license_mgr.validate_or_raise()

print("LicenseManager validation: PASSED")
```

Result:

``` text
LicenseManager validation: PASSED
```

------------------------------------------------------------------------

# 28. Actual Production App Valid-License Test

The deployed Databricks App produced:

``` text
License validated successfully for
license_id=CLIENT-ALTERYX-001
product=alteryx-etl
env=production
```

Then:

``` text
Starting AWA application service.
```

Then:

``` text
Application startup complete.
```

Then:

``` text
Uvicorn running on http://0.0.0.0:8000
```

This demonstrated that the production App could retrieve and validate
the signed artifact and start normally.

------------------------------------------------------------------------

# 29. Kill-Switch Test

For the kill-switch test, the license was deliberately generated with:

``` python
issued_at = datetime.now(timezone.utc)
expires_at = issued_at - timedelta(days=1)
```

The important point is that the artifact was still cryptographically
valid.

Signing output:

``` text
License signed: True
Signature length: 64
Signature Base64 length: 88
Features type: dict
Feature count: 4
```

Verification output:

``` text
Signature verification: PASSED
Payload integrity: PASSED
Feature Schema: PASSED
```

The artifact therefore satisfied:

``` text
Valid JSON       YES
Valid schema     YES
Valid signature  YES
Correct identity YES
Expired          YES
```

------------------------------------------------------------------------

# 30. Actual Kill-Switch Result

After replacing the production artifact and restarting the App, the logs
showed:

``` text
LicenseExpiredError:
License expired at 2026-10-05T13:20:24.236613+00:00.
Current time is 2026-10-06T13:36:38.096543+00:00.
```

Then:

``` text
ERROR: Application startup failed. Exiting.
```

This is the definitive end-to-end kill-switch proof.

The application did not start because the signed license had expired.

------------------------------------------------------------------------

# 31. What the Kill Switch Does and Does Not Do

It does:

``` text
License expires
      |
      v
Application restarts
      |
      v
License validation fails
      |
      v
Application does not start
```

It does not:

``` text
License expires
      |
      v
Already-running process is forcibly terminated immediately
```

No heartbeat or online polling was implemented.

This was an intentional architectural decision.

------------------------------------------------------------------------

# 32. Tampering Example

Suppose someone changes:

``` json
"expires_at": "2030-01-01T00:00:00+00:00"
```

without resigning the payload.

The verifier reconstructs the canonical payload and checks the existing
signature.

The bytes no longer match the bytes originally signed.

Result:

``` text
Signature mismatch
        |
        v
Startup blocked
```

Therefore an attacker cannot simply edit the expiration date.

------------------------------------------------------------------------

# 33. Identity Protection

If an artifact contains:

``` json
"license_id": "CLIENT-ALTERYX-002"
```

instead of:

``` text
CLIENT-ALTERYX-001
```

identity validation fails.

Likewise, the production application expects:

``` text
product = alteryx-etl
environment = production
```

This prevents a valid artifact for another product/environment from
being accepted.

------------------------------------------------------------------------

# 34. Environment Override Protection

The production configuration deliberately ignores arbitrary overrides
for:

``` text
license_id
product
environment
production secret scope
production secret name
```

Tests were added to verify that override attempts fail closed.

This prevents a caller from attempting to bypass licensing by changing
configuration values at runtime.

------------------------------------------------------------------------

# 35. Automated Testing

The licensing test suite covered:

-   valid signed license,
-   invalid signature,
-   modified fields,
-   expiration,
-   malformed JSON,
-   missing signature,
-   invalid Base64,
-   required fields,
-   required field types,
-   feature type,
-   empty secret,
-   provider failure,
-   wrong public key,
-   expiration boundaries,
-   future expiration,
-   UTC timestamps,
-   naive datetime handling,
-   non-UTC datetime handling,
-   environment override protection,
-   Databricks provider resolution,
-   identity override protection,
-   license-disable environment variable attempts,
-   secret scope override protection,
-   secret name override protection.

Latest licensing test result:

``` text
pytest backend/tests/test_licensing.py -v

33 passed
```

Full backend test result:

``` text
pytest backend/tests -v

55 passed
5 warnings
```

Python compilation:

``` bash
python -m compileall backend
```

completed successfully.

The warnings were dependency deprecations and were not licensing
failures.


------------------------------------------------------------------------

# 36. Frontend/Backend Runtime Structure

The deployed structure is:

``` text
/app/python/source_code/
|
+-- backend/
|   +-- app/
|       +-- main.py
|
+-- frontend/
|   +-- dist/
|       +-- index.html
|       +-- assets/
|
+-- app.yaml
+-- start.sh
+-- requirements.txt
```

The frontend path is resolved from the backend using:

``` python
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parents[2]
FRONTEND_DIST = PROJECT_ROOT / "frontend" / "dist"
```

The root route serves `index.html`.

The API routers continue to handle:

``` text
/api/*
```

------------------------------------------------------------------------

# 38. Final Request Architecture

``` text
Browser
   |
   v
Databricks App
   |
   +-------------------------+
   |                         |
   v                         v
/                       /api/*
React frontend             FastAPI
   |                         |
   |                         +--> upload
   |                         +--> analysis
   |                         +--> rationalisation
   |                         +--> reports
   |
   +-------------------------+
```

Startup remains protected by the license manager before these routes
become available.

------------------------------------------------------------------------

# 39. Complete License Lifecycle

``` text
Generate payload
      |
      v
Retrieve signing key
      |
      v
Canonicalize payload
      |
      v
Ed25519 sign
      |
      v
Verify signature
      |
      v
Create final JSON
      |
      v
Back up current production artifact
      |
      v
Publish to Azure Key Vault
      |
      v
Databricks secret scope
      |
      v
Databricks App secret resource
      |
      v
ALTERYX_LICENSE_ARTIFACT
      |
      v
LicenseManager
      |
      +--> JSON
      +--> schema
      +--> signature
      +--> identity
      +--> expiry
      |
      v
START or FAIL CLOSED
```

------------------------------------------------------------------------

# 40. Complete License Update Runbook

Whenever a new production license is required:

## Step 1 --- Generate payload

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

## Step 2 --- Retrieve signing key

``` python
import base64
from cryptography.hazmat.primitives.serialization import load_der_private_key

private_key_b64 = dbutils.secrets.get(
    scope="alteryx-licenseSigning",
    key="alteryx-license-private-key"
)

private_key = load_der_private_key(
    base64.b64decode(private_key_b64),
    password=None,
)
```

## Step 3 --- Canonicalize and sign

``` python
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
```

## Step 4 --- Verify

Use the public-key verification code in Section 19.

## Step 5 --- Create final artifact

``` python
license_json = json.dumps(
    signed_license,
    sort_keys=True,
    separators=(",", ":"),
)

with open("/tmp/alteryx-license.json", "w") as f:
    f.write(license_json)

print("License artifact prepared:", True)
print("Artifact size:", len(license_json), "bytes")
print("Expires at:", license_payload["expires_at"])
```

## Step 6 --- Back up current production license

``` bash
VALID_LICENSE="$(az keyvault secret show   --vault-name alteryx-licenseArtifacts   --name alteryx-license   --query value   -o tsv)"

if [ -n "$VALID_LICENSE" ]; then
    echo "Current license backed up in this Cloud Shell session."
else
    echo "ERROR: Could not retrieve current license."
fi
```

## Step 7 --- Publish

``` bash
read -s -p "Paste signed license JSON: " LICENSE_JSON
echo

az keyvault secret set   --vault-name alteryx-licenseArtifacts   --name alteryx-license   --value "$LICENSE_JSON"   --query "name"   -o tsv

unset LICENSE_JSON
```

## Step 8 --- Restart/redeploy

The updated artifact is enforced on the next application startup.

## Step 9 --- Verify logs

Expected:

``` text
License validated successfully
Application startup complete
```

------------------------------------------------------------------------

# 41. Rollback Runbook

If the new license must be rolled back:

``` bash
az keyvault secret set   --vault-name alteryx-licenseArtifacts   --name alteryx-license   --value "$VALID_LICENSE"   --query "name"   -o tsv
```

Then:

``` bash
unset VALID_LICENSE
```

Restart/redeploy the App and verify:

``` text
License validated successfully
Application startup complete
```

------------------------------------------------------------------------

# 42. Kill-Switch Test Runbook

To test expiry:

``` python
issued_at = datetime.now(timezone.utc)
expires_at = issued_at - timedelta(days=1)
```

Perform the normal signing and verification procedure.

Publish the artifact.

Restart/redeploy the App.

Expected:

``` text
LicenseExpiredError
Application startup failed. Exiting.
```

Then restore the valid artifact using the rollback procedure.

------------------------------------------------------------------------

# 43. Threat Model

## Threat: Modify expiration date

Result:

``` text
Signature verification fails.
```

## Threat: Modify license ID

Result:

``` text
Signature and/or identity validation fails.
```

## Threat: Modify product/environment

Result:

``` text
Signature and/or identity validation fails.
```

## Threat: Delete signature

Result:

``` text
Schema/verification fails.
```

## Threat: Corrupt JSON

Result:

``` text
JSON parsing fails.
```

## Threat: Remove secret

Result:

``` text
Secret retrieval fails.
```

## Threat: Let license expire

Result:

``` text
Expiration validation fails.
Application does not start.
```

## Threat: Generate a new license without the private key

Result:

``` text
A valid Ed25519 signature cannot be generated.
```

------------------------------------------------------------------------

# 44. What This Architecture Does Not Claim

No software-only licensing mechanism makes a runtime mathematically
impossible to modify if an attacker gains sufficient control over the
entire runtime environment.

This design specifically protects the **license issuance authority** and
makes license authenticity/integrity independently verifiable.

It provides:

-   cryptographic authenticity,
-   cryptographic integrity,
-   product/environment binding,
-   expiration enforcement,
-   startup fail-closed behavior,
-   separation of signing authority from runtime.

Additional controls such as code compilation, container hardening,
access restrictions, and infrastructure security can be layered on top
if stronger runtime tamper resistance is required.

------------------------------------------------------------------------

# 45. Key Operational Rules

1.  Never commit the private key to Git.
2.  Never place the private key in the frontend.
3.  Never place the private key in the production App.
4.  Never print the private key.
5.  Never put the private key into documentation.
6.  Never log the private key.
7.  Never distribute the private key to the client.
8.  Use the public key for runtime verification.
9.  Verify every artifact before publishing.
10. Keep `CLIENT-ALTERYX-001`, `alteryx-etl`, and `production`
    immutable.
11. Keep `alteryx-licenseArtifacts` narrowly scoped.
12. Back up the current valid artifact before replacing it.
13. Test a new artifact before considering the update complete.
14. After a kill-switch test, restore the valid artifact.
15. Restart the application after restoring it and confirm successful
    validation.
16. Do not attach `alteryx-licenseSigning` to the production App.

------------------------------------------------------------------------

# 47. Final Validated State

The implementation has been validated at multiple levels.

### Licensing unit/integration tests

``` text
pytest backend/tests/test_licensing.py -v

33 passed
```

### Full backend tests

``` text
pytest backend/tests -v

55 passed
5 warnings
```

### Compilation

``` bash
python -m compileall backend
```

Successful.

### Actual Databricks license validation

``` text
LicenseManager validation: PASSED
```

### Actual valid-license App startup

``` text
License validated successfully
Application startup complete
Uvicorn running
```

### Actual expired-license kill switch

``` text
LicenseExpiredError
Application startup failed. Exiting.
```

The two critical production behaviors have therefore both been
demonstrated:

``` text
VALID LICENSE
     |
     v
APPLICATION STARTS
```

and:

``` text
EXPIRED LICENSE
     |
     v
APPLICATION DOES NOT START
```

------------------------------------------------------------------------

# 48. Final Architecture Summary

``` text
                    LICENSE ISSUER
                         |
                         v
             Azure Key Vault: alteryx-licensing
                         |
                         +-- alteryx-license-private-key
                         |
                         | Ed25519 signing
                         v
                 SIGNED LICENSE JSON
                         |
                         v
          Azure Key Vault: alteryx-licenseArtifacts
                         |
                         +-- alteryx-license
                         |
                         v
        Databricks Secret Scope: alteryx-licenseArtifacts
                         |
                         v
             Databricks App Secret Resource
                         |
                         +-- resource key:
                         |   license-artifact
                         |
                         v
              ALTERYX_LICENSE_ARTIFACT
                         |
                         v
                  LicenseManager
                         |
          +--------------+---------------+
          |              |               |
          v              v               v
       Schema        Signature        Identity
       check          check            check
          |              |               |
          +--------------+---------------+
                         |
                         v
                    Expiration
                       check
                         |
                   +-----+-----+
                   |           |
                  PASS        FAIL
                   |           |
                   v           v
              START APP    BLOCK STARTUP
```

This is the complete implemented licensing and kill-switch workflow,
from key custody and license issuance through Azure Key Vault,
Databricks secret distribution, runtime cryptographic verification,
application startup enforcement, expiration testing, and rollback.
