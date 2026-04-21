# HypothesisBase Systems Controller

HypothesisBase **systems-controller** is a distributed control stack for HypothesisBase: a **Managed Node** agent runs on machines you want to operate, **Control Node** provides a CLI to generate keys and drive workflows against those nodes, and **Systems Controller** is the coordinating service (with a bundled web UI under `SC/web`) that ties the environment together. Together they support remote command execution, workflow automation, and operational visibility across Linux and Windows hosts.

---

## Installation

Work from the repository root (`systems-controller/`).

1. **Choose binaries for your OS**
   - **Linux:** use executables under `linux/ManagedNode/`, `linux/ControlNode/`, and `linux/SystemsController/` (each folder contains the main `.bin` plus bundled shared libraries).
   - **Windows:** use `windows/ManagedNode/ManagedNode.exe`, `windows/ControlNode/ControlNode.exe`, and `windows/SystemsController/SystemsController.exe` with the accompanying `.dll` / `.pyd` files in those folders.

2. **Follow `setup.sh`**
   - `setup.sh` documents the full bootstrap flow: generating an RSA key pair with Control Node, placing the **public** key in the Managed Node `authorized_keys` directory, and keeping the **private** key where Systems Controller / workflows expect it.
   - The file also includes **Windows-oriented** example paths for `mn.json` / `sc.json`; on Linux or macOS, prefer the root-level `MN.json` and `SC.json` (see [Configuration](#configuration)).

3. **Adjust JSON configs**
   - Copy or edit `MN.json` and `SC.json` so `work_dir`, `authorized_keys_dir`, and `control_node_path` match your layout and platform (see below).

---

## Usage

Typical **Linux** flows are collected in **`commands.sh`**. From the repo root, the sequence is:

1. **Generate keys** (once per key pair):

   ```bash
   ./linux/ControlNode/ControlNode.bin gen_key_pair \
     --pub_key_path MN/authorized_keys/pub1.pem \
     --priv_key_path MN/priv_keys/priv1.pem
   ```

2. **Start the Managed Node** (agent):

   ```bash
   ./linux/ManagedNode/ManagedNode.bin --config MN.json
   ```

3. **Run a workflow** via Control Node (uses `MN/auth_config.json` for node credentials and `MN/workflow.sh` as the script):

   ```bash
   ./linux/ControlNode/ControlNode.bin run_workflow \
     --creds_path MN/auth_config.json \
     --script_path MN/workflow.sh
   ```

4. **Start Systems Controller**:

   ```bash
   ./linux/SystemsController/SystemsController.bin
   ```

   On Windows, pass your config explicitly when needed, for example: `SystemsController.exe SC.json`.

**`MN/workflow.sh`** is an example workflow (e.g. running a remote command through the managed node). **`MN/auth_config.json`** lists managed nodes (`host`, `port`, `user`, `priv_key_path`) that Control Node uses when executing workflows.

---

## Configuration

### `MN.json` — Managed Node

Configures the agent process that listens for signed control traffic.

| Field | Description |
|--------|-------------|
| `host` | Bind address (e.g. `0.0.0.0` for all interfaces). |
| `port` | TCP port the Managed Node listens on. |
| `work_dir` | Working directory for runtime state and artifacts (e.g. `MN/work`). |
| `authorized_keys_dir` | Directory containing trusted **public** keys (e.g. `MN/authorized_keys`). |

Example (as shipped):

```json
{
  "host": "0.0.0.0",
  "port": 8000,
  "work_dir": "MN/work",
  "authorized_keys_dir": "MN/authorized_keys"
}
```

### `SC.json` — Systems Controller

Configures the Systems Controller service and how it reaches Control Node.

| Field | Description |
|--------|-------------|
| `host` | Bind address for the controller API / UI. |
| `port` | TCP port for the controller. |
| `work_dir` | Controller workspace (workflows, auth configs, etc.), e.g. `SC`. |
| `control_node_path` | Path to the Control Node executable (e.g. `linux/ControlNode/ControlNode.bin` or `windows/ControlNode/ControlNode.exe`). |

Example (as shipped):

```json
{
  "host": "localhost",
  "port": 8001,
  "work_dir": "SC",
  "control_node_path": "linux/ControlNode/ControlNode.bin"
}
```

Ensure the public key from `gen_key_pair` is deployed under the Managed Node’s `authorized_keys_dir`, and the matching private key path is consistent with `MN/auth_config.json` and your operational layout (as described in `setup.sh`).

---

## Folder structure

| Path | Purpose |
|------|---------|
| **`MN/`** | **Managed Node** side: `authorized_keys/`, `priv_keys/`, `work/` (runtime output such as logs and maps), workflow scripts (`workflow.sh`, …), and `auth_config.json` for Control Node. |
| **`SC/`** | **Systems Controller** workspace: `web/` (static UI build), `workspace/workflows/`, `workspace/auth_configs/`, and other controller-local state under `work_dir`. |
| **`linux/`** | Packaged **Linux** builds: `ManagedNode/`, `ControlNode/`, `SystemsController/` — each contains the main binary and bundled `.so` dependencies. |
| **`windows/`** | Packaged **Windows** builds: same three components as `.exe` plus `.dll` / `.pyd` support files. |

Root-level **`MN.json`** / **`SC.json`** are the default-style configs for the shipped layout; **`commands.sh`** and **`setup.sh`** document concrete commands and cross-platform path examples.

---

## Requirements

- **Shell:** Bash (or Git Bash on Windows) to run the documented command lines.
- **OS/arch:** Use the `linux/` or `windows/` tree that matches your deployment; binaries are prebuilt and self-contained with vendored native libraries where applicable.
- **Networking:** Managed Node and Systems Controller ports (`MN.json` / `SC.json`) must be reachable from the components that connect to them.
- **Permissions:** Executables need execute permission on Unix; private keys should be readable only by the operator account running Control Node / Systems Controller.

No separate language runtime (Python, Node, etc.) is required to **run** the shipped binaries—only the correct platform folder and valid JSON configuration.

---

## Contributing

Contributions are welcome.

1. **Issues:** Open an issue to describe bugs, desired features, or documentation gaps (include OS, config snippets, and steps to reproduce when reporting problems).
2. **Changes:** Fork the repository, create a focused branch, and submit a pull request with a clear summary of behavior changes and any config or operational impact.
3. **Scope:** Keep changes minimal and consistent with existing layout (`MN/`, `SC/`, `linux/`, `windows/`) unless the maintainers agree on a broader refactor.

For maintainers: keep `setup.sh` and `commands.sh` aligned with real invocation patterns when behavior or paths change.

