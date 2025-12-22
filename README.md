# Hubble Simple Application Reference for Zephyr

This repository is a minimal Zephyr application that demonstrates the Hubble BLE advertisement flow used by Hubble devices. It includes a GitHub Actions workflow that can build the application for a target board and produce a password-protected archive containing the built artifacts. You can either:

* Use the **GitHub Actions** `Build Application` workflow to generate password-protected binaries via a manual `workflow_dispatch` run (recommended for non-developers), or
* Build the application locally with **west**.

> **Important:** To register devices or to use Hubble services you **must** obtain your organization ID and API token from Hubble’s website. See the **Hubble credentials** section below.


## Quick links

* Workflow: `.github/workflows/build.yml` — the `Build Application` workflow is triggered via `workflow_dispatch` and accepts the inputs described below.
* Local build instructions: `west` build and flash steps are described below and in the repository README.


## Using the GitHub Actions **Build Application** workflow

The repository contains a GitHub Actions workflow named **Build Application** (`.github/workflows/build.yml`). It is configured with `workflow_dispatch` inputs so you can manually run the workflow from the GitHub UI or with the GitHub CLI.

### Workflow inputs

When running the workflow manually you will be asked to provide the following inputs (the workflow will also fall back to repository secrets when available):

* `board` (required) — the Zephyr board to build for (e.g., `nrf52840dk/nrf52840`).
* `org_id` (optional) — your Hubble Organization ID (overrides secret). If set, `api_token` is required.
* `api_token` (optional) — API token for your org (requires `org_id` to be set).
* `device_key` (optional) — supply an existing device key to **bypass** registration.
* `device_name` (optional) — friendly device name (if registering). If empty, the workflow generates a name.
* `artifact_encryption_pw` (optional) — password used to encrypt the output archive. If not provided, the workflow will attempt to use the `HUBBLE_ARTIFACT_PASSWORD` repository secret. **A password is required**, because the produced package contains private keys.

**Repository secrets used by the workflow (optional):**

* `HUBBLE_ORG_ID` — default org ID used when `org_id` input is not provided.
* `HUBBLE_API_TOKEN` — default API token used when `api_token` input is not provided.
* `HUBBLE_ARTIFACT_PASSWORD` — default artifact encryption password used when `artifact_encryption_pw` is not provided.

> **Note:** If you plan to fork this repository and run the workflow from your fork, you need to create these secrets in *your fork* via **Settings → Secrets and variables → Actions** (or supply the values as workflow inputs during dispatch).

### What the workflow does

High-level steps (see full workflow for details):

1. Checks out the repository and installs small helper tools.
2. Sets up Zephyr via the official `zephyrproject-rtos/action-zephyr-setup` action.
3. If a `device_key` input is not provided, the workflow **registers** a device using the Hubble API endpoints (production or sandbox) and exports `HUBBLE_DEVICE_KEY`/`HUBBLE_DEVICE_ID`. The registration step requires valid org credentials (inputs or secrets).
4. Builds the Zephyr application via `west build -p -b "$BOARD" .` — **this embeds the device key** into the built binary in the same way the repository example does. See the security note below.
5. Packages the `./build` directory into a password-protected ZIP using AES-256 and uploads the archive as a GitHub Actions artifact. The ZIP filename is of the form `<<sanitized-board>>-build-<run_id>.zip`, and the artifact name is `hubble-simple-app-<device-name>`.

### Running the workflow (GitHub UI)

1. In the repository, click the **Actions** tab.
2. Select **Build Application** (or open `.github/workflows/build.yml`).
3. Click **Run workflow**.
4. Enter the required input `board` and any optional inputs (`org_id`, `api_token`, `device_key`, `device_name`, `artifact_encryption_pw`).

   * If you want the workflow to register a device on your behalf, provide `org_id` and `api_token` (or make sure secrets `HUBBLE_ORG_ID` / `HUBBLE_API_TOKEN` are set).
   * If you provide `device_key` the registration step is skipped.
5. Click **Run workflow**. The workflow will run and you can monitor it in the Actions UI.

### Running the workflow (GitHub CLI)

You can dispatch the workflow from the command line with `gh`:

```bash
# Example: builds for nrf52840dk/nrf52840 and supplies the required encryption password
gh workflow run .github/workflows/build.yml \
  -f board=nrf52840dk/nrf52840 \
  -f org_id=<YOUR_ORG_ID> \
  -f api_token=<YOUR_API_TOKEN> \
  -f device_name=my-test-device \
  -f artifact_encryption_pw=<super-secret-password>
```

(If you prefer not to include credentials on the command line, omit `org_id`/`api_token` and instead set `HUBBLE_ORG_ID` / `HUBBLE_API_TOKEN` in your repo secrets.)

### Downloading and decrypting artifacts

1. After the workflow finishes, open the run in the **Actions** UI and find the **Artifacts** section (right side or bottom). Click the `hubble-simple-app-<device-name>` artifact to download.
2. The artifact contains a single AES-256 password-protected ZIP named like:

   ```
   <sanitized-board>-build-<run_id>.zip
   ```

   The password you must use is either:

   * The `artifact_encryption_pw` you supplied when dispatching the workflow, **or**
   * The repository secret `HUBBLE_ARTIFACT_PASSWORD` (if the workflow used that secret).
3. To extract the ZIP use `7z` (recommended, matches the way the ZIP was created):

```bash
# Linux / macOS (p7zip / 7z)
7z x <sanitized-board>-build-<run_id>.zip -p<password>

# Windows (7-Zip GUI or CLI)
# GUI: right-click → 7-Zip → Extract files… and supply the password
# CLI (PowerShell/CMD) with 7z on PATH:
7z x <sanitized-board>-build-<run_id>.zip -p<password>
```

> The workflow encrypts the archive with AES-256 using `7z` so use a compatible extractor (7-Zip / p7zip).


## Building locally with **west**

If you prefer to build locally, follow these steps. These match the steps used by the workflow to build the application.

1. Prepare a Zephyr development environment. See the official Zephyr getting started guide for the platform you are using. The repository README gives the minimal steps to get the Zephyr workspace setup.

2. Create a workspace and clone this repo (example):

```bash
mkdir hubble-workspace
cd hubble-workspace
git clone https://github.com/HubbleNetwork/hubble-reference-zephyr-simple
```

3. (Optional) Create and activate a Python venv:

```bash
python -m venv .venv
source .venv/bin/activate
```

4. Initialize west and update modules:

```bash
# If you don't have west installed:
pip install west

# Initialize a local west workspace for this app
west init -l hubble-reference-zephyr-simple
west update

# Export the Zephyr CMake package
west zephyr-export

# Install project python packages (if any)
west packages pip --install

# Optional: install Zephyr SDK with west
west sdk install
```

These steps are the same high-level steps used by the workflow to prepare Zephyr.

5. Build the app (local device key example):

The example application expects a device key to be available from the ```HUBBLE_DEVICE_KEY``` environment variable. If this is not set the key will default to ```1111111111111111111111111111111111111111111=```.

```bash
export HUBBLE_DEVICE_KEY=<your-device-key-here>

# Build with west (replace <BOARD> with your board - e.g. nrf52840dk/nrf52840)
west build -p -b <BOARD> .
```

6. Flash (if you have a connected board):

```bash
west flash
```

> **Note:** The workflow embeds the device key into the binary. That means any key you embed is compiled into the firmware — see the Security section below.

### `prj.conf` and important build-time configuration

This application uses a small `prj.conf` to enable the shell, Bluetooth and logging. The example `prj.conf` in this repository configures logging, Bluetooth peripheral role, power management, and defines the main stack size. You can find those settings in `prj.conf`. 

## Hubble credentials and device registration

* If you run the workflow and **do not** supply `device_key`, the workflow will attempt to **register** one device for you via Hubble's API (production or test endpoint) — this requires valid credentials (`org_id` and `api_token`) as a workflow input or as repository secrets.
* If you supply `device_key` to the workflow, the registration step will be bypassed and the workflow will use the provided key directly.

> **How to obtain `org_id` and `api_token`:** visit Hubble’s provisioning or developer portal (the Hubble website) and follow the instructions to create or retrieve your organization ID and API token. These are required for device registration via the Hubble API.


## Security considerations (important)

1. **Device keys are embedded into the firmware**: The build step embeds the Hubble device key into the application binary via `-DKEY="..."`. That means the firmware image contains the device key. If you plan to distribute binaries, be aware that anyone who can access the binary can extract the embedded key. The workflow mitigates this by packaging build artifacts into a **password-protected AES-256 ZIP**, but the firmware itself still contains the key. Please treat any such key as sensitive.

2. **Artifact encryption password**: You must supply `artifact_encryption_pw` or set `HUBBLE_ARTIFACT_PASSWORD` as a repository secret. The workflow will fail if no password is provided, because the produced archive contains private keys and must be encrypted.

3. **Recommended production approach**: For production systems do **not** compile secrets into the firmware image. Use a secure provisioning model (PSA Crypto, secure element, or secure keystore) and avoid embedding long-term secrets into build artifacts. The reference application includes a convenient embedded key path only for demonstration and testing. See `src/main.c` where the example decodes an embedded base64 key; this implementation is for demonstration purposes only and should be replaced with a proper secure key retrieval mechanism for production.


## Troubleshooting & tips

* If the workflow fails during device registration, ensure your `org_id` and `api_token` are correct and have device-registration permissions. The workflow first tries the production endpoint and then falls back to the sandbox endpoint if production fails.
* If you run the workflow from a **fork**, repository secrets from the upstream repo are **not** copied to the fork. Add the required secrets to your fork or provide all sensitive inputs at dispatch time.
* If the built artifact is missing, check that the `7z` archiver step completed successfully — the workflow uses `7z` and will fail if `7z` is unavailable. The workflow installs `p7zip-full` at the start to ensure extraction compatibility across platforms.


## License

This repository is licensed under the Apache 2.0 license (see `LICENSE`).
