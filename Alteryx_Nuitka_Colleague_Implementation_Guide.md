# Alteryx — Complete Nuitka Protected Deployment Guide

## Purpose

This guide is written for a developer who has **not previously performed the Nuitka build** for this project.

The goal is to create a Databricks Apps deployment artifact in which the proprietary Python implementation is compiled into native Linux extension modules.

The final deployment must:

- run on the Databricks Apps Python 3.11 runtime;
- contain the compiled `awa` and `backend.app` packages;
- contain required non-Python runtime files such as `grammar.lark`;
- contain the React `frontend/dist` build;
- contain no proprietary `.py` implementation files;
- contain no `.pyc` files or `__pycache__`;
- contain no `.env` files;
- contain no individual file larger than 10 MB;
- start FastAPI/Uvicorn on the Databricks-provided port.

> **Important:** Do not change the working `Alteryx-2` application while performing this build. Perform the protected build in a separate copy/workspace.

---

# 1. Expected Project Structure

Before starting, the project should look approximately like this:

```text
Alteryx-2/
├── backend/
│   ├── app/
│   ├── artifacts/
│   ├── awa/
│   ├── tests/
│   ├── __init__.py
│   └── requirements.txt
├── docs/
├── frontend/
│   └── dist/
├── fixtures/
├── tests/
├── requirements.txt
├── pyproject.toml
└── ...
```

The important source directories are:

```text
backend/awa/
backend/app/
frontend/dist/
backend/artifacts/
```

The `backend/awa` package contains the proprietary AWA implementation.

The `backend/app` package contains the FastAPI application.

---

# 2. Use a Separate Build Copy

Create a separate copy of the project for the protected build.

For example:

```text
Alteryx-2
Alteryx-2-production
```

Do **not** delete or modify the source files in the working application until the protected build has been independently validated.

---

# 3. Open the Databricks Build Environment

The build must be performed in a Linux x86-64 environment compatible with Databricks Apps.

The tested environment used:

```text
Python 3.11.11
Linux x86-64
Nuitka 4.2.2
```

The Databricks Apps runtime used for deployment was Python 3.11.

This is important because a binary compiled for Python 3.12 cannot be used as a Python 3.11 extension module.

---

# 4. Verify Python

Run this complete command:

```bash
python3 --version
```

The output must show Python 3.11.x.

If `python3` is not Python 3.11, do not continue with the build.

---

# 5. Create a Python 3.11 Virtual Environment

Run this complete command:

```bash
python3 -m venv /tmp/nuitka311
```

Activate it:

```bash
source /tmp/nuitka311/bin/activate
```

Verify the Python version:

```bash
python --version
```

Expected:

```text
Python 3.11.x
```

---

# 6. Install Nuitka

With the virtual environment activated, run:

```bash
python -m pip install --upgrade pip
```

Then install the tested Nuitka version:

```bash
python -m pip install "Nuitka==4.2.2"
```

Verify it:

```bash
python -m nuitka --version
```

The version should be:

```text
4.2.2
```

---

# 7. Install the Project Dependencies

From the **root of the Alteryx project**, run:

```bash
python -m pip install -r requirements.txt
```

If the backend has its own requirements file and it is required by the build environment, run:

```bash
python -m pip install -r backend/requirements.txt
```

The production requirements used by the application include packages such as:

```text
fastapi
uvicorn
pydantic
python-multipart
python-dotenv
networkx
lark
click
python-docx
pillow
openpyxl
pandas
numpy
```

---

# 8. Go to the Project Root

Before running the Nuitka commands, make sure the terminal is in the project root.

Run:

```bash
pwd
```

Then run:

```bash
ls
```

You should see files/directories such as:

```text
backend
frontend
requirements.txt
pyproject.toml
```

If `backend` is not present, stop. You are not in the project root.

---

# 9. Create the Deployment Build Directories

Run:

```bash
mkdir -p deployment/awa_package_py311
```

Then:

```bash
mkdir -p deployment/app_package_py311
```

---

# 10. Compile the AWA Package

This is the complete command. Copy it exactly:

```bash
PYTHONPATH=backend /tmp/nuitka311/bin/python -m nuitka \
  --mode=package \
  --include-package=awa \
  backend/awa \
  --output-dir=deployment/awa_package_py311 \
  --remove-output
```

### What this does

It compiles:

```text
backend/awa/
```

into a Nuitka-compiled package.

The important output is:

```text
deployment/awa_package_py311/awa.cpython-311-x86_64-linux-gnu.so
```

There should also be a generated `.pyi` file.

The compiled AWA package includes the implementation of the Python modules without delivering the original `.py` implementation files.

---

# 11. Verify the AWA Build

Run:

```bash
ls -lh deployment/awa_package_py311/
```

You should see a file similar to:

```text
awa.cpython-311-x86_64-linux-gnu.so
```

The tested build produced an AWA binary of approximately 8 MB.

Do not rename the `.so` file.

The filename contains:

```text
cpython-311
```

because it is compiled for Python 3.11.

---

# 12. Compile the FastAPI Application Package

Now compile the `backend.app` package.

Run this complete command:

```bash
PYTHONPATH=. /tmp/nuitka311/bin/python -m nuitka \
  --mode=package \
  --include-package=backend.app \
  backend/app \
  --output-dir=deployment/app_package_py311 \
  --remove-output
```

### What this does

It compiles:

```text
backend/app/
```

into a Nuitka package.

The important output is:

```text
deployment/app_package_py311/app.cpython-311-x86_64-linux-gnu.so
```

There should also be a generated `.pyi` file.

---

# 13. Verify the Application Build

Run:

```bash
ls -lh deployment/app_package_py311/
```

You should see:

```text
app.cpython-311-x86_64-linux-gnu.so
```

The tested build produced an application binary of approximately 2–3 MB.

---

# 14. Important: Copy Required Runtime Data

Nuitka compilation does not automatically mean that every non-Python data file is embedded into the compiled package.

The application requires:

```text
grammar.lark
```

Copy it into the protected deployment root.

Run:

```bash
cp backend/awa/expressions/grammar.lark deployment/
```

Verify:

```bash
ls -lh deployment/grammar.lark
```

The file must exist.

---

# 15. Build the Protected Databricks App Directory

Create the final deployment directory:

```bash
rm -rf deployment/final-databricks-app
```

Then:

```bash
mkdir -p deployment/final-databricks-app/backend
```

Then:

```bash
mkdir -p deployment/final-databricks-app/frontend
```

Then:

```bash
mkdir -p deployment/final-databricks-app/artifacts
```

Then:

```bash
mkdir -p deployment/final-databricks-app/docs
```

---

# 16. Copy the Compiled FastAPI Application

Run:

```bash
cp deployment/app_package_py311/app.cpython-311-x86_64-linux-gnu.so deployment/final-databricks-app/backend/
```

---

# 17. Create the Backend Package Marker

The final deployment needs this package marker:

```text
deployment/final-databricks-app/backend/__init__.py
```

Create it with:

```bash
touch deployment/final-databricks-app/backend/__init__.py
```

This file is only a package marker and does not contain proprietary implementation logic.

---

# 18. Copy the Compiled AWA Package

Run:

```bash
cp deployment/awa_package_py311/awa.cpython-311-x86_64-linux-gnu.so deployment/final-databricks-app/
```

---

# 19. Copy the AWA Type Stub

Run:

```bash
cp deployment/awa_package_py311/awa.pyi deployment/final-databricks-app/
```

The `.pyi` file is a type/interface stub, not the original implementation.

---

# 20. Copy the Grammar File

Run:

```bash
cp backend/awa/expressions/grammar.lark deployment/final-databricks-app/
```

The final directory should now contain:

```text
deployment/final-databricks-app/
├── backend/
│   ├── __init__.py
│   └── app.cpython-311-x86_64-linux-gnu.so
├── awa.cpython-311-x86_64-linux-gnu.so
├── awa.pyi
└── grammar.lark
```

---

# 21. Copy Runtime Artifacts

The application uses runtime artifacts under:

```text
backend/artifacts/
```

Copy them:

```bash
cp -R backend/artifacts/. deployment/final-databricks-app/artifacts/
```

This preserves files required at runtime, such as the LLM cache.

---

# 22. Copy the React Frontend

The frontend must already be built.

Verify:

```bash
ls -lh frontend/dist/
```

You should see:

```text
index.html
assets/
```

Copy the complete React build:

```bash
cp -R frontend/dist deployment/final-databricks-app/frontend/
```

The final location must therefore be:

```text
deployment/final-databricks-app/frontend/dist/
```

---

# 23. Copy the Tool Support Documentation

If the application expects the tool support matrix, copy it:

```bash
cp docs/tool-support-matrix.md deployment/final-databricks-app/docs/
```

---

# 24. Copy Requirements

Copy the root requirements file:

```bash
cp requirements.txt deployment/final-databricks-app/
```

---

# 25. Create app.yaml

Create:

```text
deployment/final-databricks-app/app.yaml
```

Its complete contents must be:

```yaml
command:
  - "/bin/bash"
  - "./start.sh"
```

Do not use a source Python entry point such as:

```text
python backend/app/main.py
```

The compiled application must be loaded through the package import.

---

# 26. Create start.sh

Create:

```text
deployment/final-databricks-app/start.sh
```

Put the following complete contents into it:

```bash
#!/usr/bin/env bash
set -e

ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$ROOT"

export PYTHONPATH="$ROOT:$ROOT/backend"

HOST="0.0.0.0"
PORT="${DATABRICKS_APP_PORT:-8000}"

echo "========================================"
echo " STARTING PROTECTED AWA APPLICATION"
echo "========================================"
echo "ROOT: $ROOT"
echo "HOST: $HOST"
echo "PORT: $PORT"
echo "========================================"

exec python -m uvicorn backend.app.main:app \
    --host "$HOST" \
    --port "$PORT"
```

---

# 27. Make start.sh Executable

Run:

```bash
chmod +x deployment/final-databricks-app/start.sh
```

Verify:

```bash
ls -l deployment/final-databricks-app/start.sh
```

It should have execute permission.

---

# 28. Final Expected Directory

At this point the deployment should look approximately like:

```text
deployment/final-databricks-app/
├── app.yaml
├── start.sh
├── requirements.txt
├── awa.cpython-311-x86_64-linux-gnu.so
├── awa.pyi
├── grammar.lark
├── backend/
│   ├── __init__.py
│   └── app.cpython-311-x86_64-linux-gnu.so
├── artifacts/
│   └── ...
├── frontend/
│   └── dist/
│       ├── index.html
│       └── assets/
│           ├── ...
└── docs/
    └── tool-support-matrix.md
```

There must **not** be:

```text
backend/app/*.py
backend/awa/*.py
__pycache__/
*.pyc
.env
```

in the final deployment.

---

# 29. Security / Source Audit

Run this command from the project root:

```bash
find deployment/final-databricks-app -type f \( -name "*.py" -o -name "*.pyc" -o -name ".env" \) -print
```

For the protected build, this should return only the intentional package marker:

```text
deployment/final-databricks-app/backend/__init__.py
```

No proprietary implementation `.py` files should appear.

Now check for Python cache directories:

```bash
find deployment/final-databricks-app -type d -name "__pycache__" -print
```

This should return nothing.

---

# 30. Check for Oversized Files

Databricks Apps has a 10 MB per-file limit.

Run:

```bash
find deployment/final-databricks-app -type f -size +10M -print
```

This should return nothing.

If this command prints any file, **do not deploy yet**.

---

# 31. Check for Environment Files

Run:

```bash
find deployment/final-databricks-app -type f -name ".env" -print
```

This should return nothing.

Do not copy the local `.env` file into the deployment package.

---

# 32. Test the Protected Python Packages

Before deploying to Databricks, test that the compiled packages can be imported.

Run:

```bash
cd deployment/final-databricks-app
```

Then:

```bash
PYTHONPATH="$PWD:$PWD/backend" python -c "import awa; print('AWA IMPORT: PASS')"
```

Expected:

```text
AWA IMPORT: PASS
```

Then:

```bash
PYTHONPATH="$PWD:$PWD/backend" python -c "from backend.app.main import app; print('FASTAPI IMPORT: PASS'); print('ROUTES:', len(app.routes))"
```

Expected:

```text
FASTAPI IMPORT: PASS
ROUTES: 11
```

The exact route count may change if the application is intentionally modified later; the tested protected build had 11 routes.

---

# 33. Test the Protected AWA Parser

Run:

```bash
PYTHONPATH="$PWD:$PWD/backend" python -c "from awa.expressions.parser import parse_expression; print(parse_expression('[Revenue] > 100')); print('PROTECTED AWA PARSE: PASS')"
```

This test verifies that:

1. the compiled AWA package loads;
2. the parser loads;
3. `grammar.lark` is available;
4. the expression parser can execute.

---

# 34. Start the Protected Application Locally

From:

```text
deployment/final-databricks-app
```

run:

```bash
DATABRICKS_APP_PORT=8000 ./start.sh
```

The application should start without importing the original backend Python source.

You should see Uvicorn start on:

```text
http://0.0.0.0:8000
```

Keep this terminal running.

---

# 35. Test the API

Open another terminal.

Run:

```bash
curl -i http://127.0.0.1:8000/api/config
```

The expected HTTP status is:

```text
HTTP/1.1 200 OK
```

The tested response was similar to:

```json
{
  "code_based_workflows_url": null,
  "kpi_ontology_bank_url": null,
  "email_from_address": "noreply@awa-etl.internal"
}
```

---

# 36. Test the Frontend

Run:

```bash
curl -i http://127.0.0.1:8000/
```

Expected:

```text
HTTP/1.1 200 OK
```

Then:

```bash
curl -i http://127.0.0.1:8000/index.html
```

Expected:

```text
HTTP/1.1 200 OK
```

Then check a frontend asset:

```bash
find frontend/dist/assets -type f -maxdepth 1 -print
```

Copy one returned asset path into a curl request.

For example, if the asset is:

```text
frontend/dist/assets/index-CnBeFMZH.js
```

run:

```bash
curl -i http://127.0.0.1:8000/assets/index-CnBeFMZH.js
```

Expected:

```text
HTTP/1.1 200 OK
```

The tested protected build returned HTTP 200 for:

```text
/
 /index.html
 /api/config
 /assets/index-CnBeFMZH.js
```

---

# 37. LLM Configuration

Do **not** put the Azure LLM key inside the protected deployment directory.

Do not copy:

```text
.env
```

into the Databricks App.

The application reads the following environment variables:

```text
AZURE_ENDPOINT
AZURE_LLAMAKEY
AZURE_DEPLOYMENT
AZURE_DEPLOYMENT_NAME
AWA_LLM_TEMPERATURE
AWA_LLM_TIMEOUT
AWA_LLM_BUSINESS_REPORT_TIMEOUT_SECONDS
AWA_LLM_RETRY_ATTEMPTS
AWA_LLM_MAX_TOKENS
AWA_LLM_ENABLED
```

The tested Azure deployment configuration used:

```text
AZURE_ENDPOINT=https://llama--resource.services.ai.azure.com/openai/v1
AZURE_DEPLOYMENT=Llama-4-Maverick-17B-128E-Instruct-FP8
AZURE_DEPLOYMENT_NAME=Llama-4-Maverick-17B-128E-Instruct-FP8
AWA_LLM_TEMPERATURE=0.0
AWA_LLM_TIMEOUT=30.0
AWA_LLM_BUSINESS_REPORT_TIMEOUT_SECONDS=60.0
AWA_LLM_RETRY_ATTEMPTS=2
AWA_LLM_MAX_TOKENS=500
AWA_LLM_ENABLED=true
```

The API key:

```text
AZURE_LLAMAKEY
```

must be provided through the Databricks secret configuration.

Never place the actual key in:

```text
app.yaml
start.sh
requirements.txt
the deployment folder
Git
the compiled source package
```

---

# 38. Databricks App Configuration

The final directory to deploy is:

```text
deployment/final-databricks-app
```

The Databricks App should use:

```text
Python 3.11
```

The application command is already defined by:

```text
app.yaml
```

which runs:

```text
./start.sh
```

The script automatically uses:

```text
DATABRICKS_APP_PORT
```

when Databricks provides it.

Do not hard-code the Databricks production port.

---

# 39. Deploy the Protected Artifact

Upload/deploy the contents of:

```text
deployment/final-databricks-app
```

to the Databricks App source location.

The deployed App must contain the compiled `.so` files and runtime assets, not the original proprietary source tree.

---

# 40. Post-Deployment Checks

After deployment, verify:

### Application status

The Databricks App should show:

```text
Running
```

### Root page

Open the App URL and verify the React application loads.

### API

Verify:

```text
/api/config
```

returns HTTP 200.

### Frontend assets

Verify the React UI loads without JavaScript 404 errors.

### Logs

The logs should show Uvicorn starting successfully.

---

# 41. LLM Post-Deployment Check

The LLM environment must be configured before testing LLM-dependent functionality.

The application should receive:

```text
AZURE_ENDPOINT
AZURE_LLAMAKEY
AZURE_DEPLOYMENT
AZURE_DEPLOYMENT_NAME
AWA_LLM_ENABLED=true
```

If the application log says:

```text
configuration: unavailable
fallback: deterministic
```

then the Azure LLM configuration has not been successfully supplied to the runtime.

Do not rebuild Nuitka binaries merely because the LLM configuration is missing. The LLM configuration is supplied at runtime through environment/secret configuration.

---

# 42. What Must Never Be Put in the Final Artifact

Do not deploy these:

```text
.env
.env.example
backend/app/*.py
backend/awa/*.py
*.pyc
__pycache__/
private signing keys
Azure LLM API keys
development-only credentials
```

The final deployment is intended to contain the compiled implementation plus the runtime resources required by the application.

---

# 43. Final Build Checklist

Before giving the artifact to the deployment team, verify every item below.

### Build

- [ ] Python 3.11 used
- [ ] Nuitka 4.2.2 used
- [ ] AWA package compiled
- [ ] FastAPI application package compiled
- [ ] `grammar.lark` copied
- [ ] React `frontend/dist` copied
- [ ] runtime artifacts copied
- [ ] `app.yaml` created
- [ ] `start.sh` created
- [ ] `start.sh` executable

### Source protection

- [ ] No proprietary `.py` files
- [ ] No `.pyc` files
- [ ] No `__pycache__`
- [ ] No `.env`
- [ ] No API keys
- [ ] No private signing keys

### Databricks compatibility

- [ ] Python 3.11 binary
- [ ] Linux x86-64 build
- [ ] No file larger than 10 MB
- [ ] Databricks App port comes from `DATABRICKS_APP_PORT`

### Runtime

- [ ] `import awa` passes
- [ ] `from backend.app.main import app` passes
- [ ] AWA parser test passes
- [ ] FastAPI starts
- [ ] `/api/config` returns 200
- [ ] `/` returns 200
- [ ] `/index.html` returns 200
- [ ] frontend assets return 200
- [ ] React UI loads

### LLM

- [ ] Azure endpoint configured
- [ ] Azure deployment configured
- [ ] Azure deployment name configured
- [ ] LLM API key supplied through secret configuration
- [ ] `AWA_LLM_ENABLED=true`
- [ ] LLM functionality tested after secret configuration

---

# 44. Final Protected Artifact

The artifact that should ultimately be supplied for Databricks deployment is:

```text
deployment/final-databricks-app/
```

The critical compiled components are:

```text
awa.cpython-311-x86_64-linux-gnu.so
backend/app.cpython-311-x86_64-linux-gnu.so
```

The tested build demonstrated that the compiled packages can:

- import successfully;
- load the AWA parser;
- load `grammar.lark`;
- start FastAPI/Uvicorn;
- serve the React frontend;
- serve API routes;
- return HTTP 200 for the tested frontend/API requests.

The original proprietary Python implementation is not intentionally included in the final protected deployment artifact.
