# B85 CCI - Ansible Automation Platform Execution Environment

A custom Ansible Automation Platform (AAP) Execution Environment that bundles Hitachi Command Control Interface (CCI/HORCM) binaries to enable execution of `raidcom` commands against Hitachi storage arrays.

## Overview

This repository builds a containerized Ansible Execution Environment based on CentOS Stream 9 that includes:

- **Ansible Core** and **Ansible Runner** for orchestration
- **HORCM (Hitachi RAID Manager)** binaries for storage array management
- **Hitachi VSP One Block collection** for Ansible playbooks
- System dependencies required for HORCM installation and operation

The resulting container can be used with Ansible Automation Platform (AAP) or standalone for executing storage automation tasks against Hitachi storage systems.

## Prerequisites (macOS)

### 1. Install Homebrew (if not already installed)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. Install Python via Homebrew

macOS ships with a system Python that should not be modified. Install Python via Homebrew instead:

```bash
brew install python
```

### 3. Install Ansible Builder in a virtual environment

Using a virtual environment avoids conflicts with the system Python and Homebrew-managed packages:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install ansible-builder
```

> **Tip**: Activate the virtual environment (`source .venv/bin/activate`) each time you open a new terminal before running `ansible-builder`.

### 4. Install and start a container runtime

**Option A — Docker Desktop (recommended):**

```bash
brew install --cask docker
```

After installation, launch the **Docker Desktop** app from Applications and wait until the Docker icon in the menu bar shows "Docker Desktop is running" before proceeding with the build.

**Option B — Podman:**

```bash
brew install podman
podman machine init
podman machine start
```

### 5. Apple Silicon (M-series) — platform requirement

> **Important**: HORCM binaries are compiled for **Linux x86_64 only**. On Apple Silicon Macs (M1/M2/M3/M4), you must instruct Docker (or Podman) to build for the `linux/amd64` platform.

Set this environment variable before every `ansible-builder` invocation:

```bash
export DOCKER_DEFAULT_PLATFORM=linux/amd64
```

Add it to your shell profile (`~/.zshrc`) to make it permanent:

```bash
echo 'export DOCKER_DEFAULT_PLATFORM=linux/amd64' >> ~/.zshrc
source ~/.zshrc
```

### 6. Prepare HORCM.tgz

Obtain the `RMHORC` cpio archive from Hitachi, then run the following commands to produce `HORCM.tgz`.

> **Note on macOS differences vs Linux**:
> - `sudo` is required for `chown root:...` on macOS.
> - The `COPYFILE_DISABLE=1` environment variable prevents macOS from embedding extended attributes (resource forks, `.DS_Store`) inside the tarball, which would corrupt the archive when extracted on Linux.

```bash
cat RMHORC | cpio -ivdum
chmod -R 777 ./HORCM
sudo chown -R root:sys ./HORCM
COPYFILE_DISABLE=1 tar -cvzpf HORCM.tgz ./HORCM
```

Place the resulting `HORCM.tgz` file in the repository root directory. It is referenced in `execution_environment.yml` and will be copied into the container during the build.

## Project Structure

```
.
├── README.md                          # This file
├── execution_environment.yml          # Ansible Builder configuration
├── requirements.yml                   # Ansible Galaxy collections/roles
├── requirements.txt                   # Python package dependencies
├── bindep.txt                         # System package dependencies
├── horcm100.conf.j2                   # HORCM configuration template
├── HORCM.tgz                          # HORCM binaries (must be added)
└── context/
    ├── Containerfile                  # Generated Docker build file
    └── _build/                        # Build artifacts (generated)
```

## Configuration Files

### `execution_environment.yml`
Defines the Execution Environment specification:
- Base image: `quay.io/centos/centos:stream9`
- Python dependencies
- System dependencies
- Ansible collections
- Custom build steps for HORCM installation

### `requirements.yml`
Specifies Ansible Galaxy collections required:
- `hitachivantara.vspone_block` - Hitachi VSP One Block collection for managing storage

### `bindep.txt`
System-level dependencies installed via the package manager:
- `python3` - Python runtime
- `findutils` - Utilities for searching files
- `tar` - Archive tool for extracting HORCM binaries

## Building the Execution Environment (macOS)

### Step 1: Activate the virtual environment

```bash
source .venv/bin/activate
```

### Step 2: Confirm HORCM.tgz is present

```bash
ls -la HORCM.tgz
```

### Step 3: Set the target platform (Apple Silicon)

Skip this step only if you are on an Intel Mac.

```bash
export DOCKER_DEFAULT_PLATFORM=linux/amd64
```

### Step 4: Build the Container Image

```bash
ansible-builder build -t my-ee:latest --container-runtime docker
```

**Options:**
- `-t, --tag` - Tag for the built image (e.g., `localhost/my-ee:1.0`)
- `-v` / `--verbosity` - Increase verbosity (e.g., `-v 3`)
- `--container-runtime` - Explicitly pass `docker` or `podman` (recommended on macOS)

**Example with full options (Docker):**

```bash
ansible-builder build \
  -t quay.io/myregistry/hitachi-ee:1.0 \
  -v 3 \
  --container-runtime docker
```

**Example using Podman:**

```bash
ansible-builder build \
  -t quay.io/myregistry/hitachi-ee:1.0 \
  -v 3 \
  --container-runtime podman
```

> **Apple Silicon + Podman**: Podman on macOS uses a Linux VM via `podman machine`. The `DOCKER_DEFAULT_PLATFORM` variable is Docker-specific. For Podman, pass the platform via the build argument:
>
> ```bash
> BUILDAH_FORMAT=docker ansible-builder build \
>   -t my-ee:latest \
>   --container-runtime podman \
>   --build-arg BUILDPLATFORM=linux/amd64
> ```

### Step 5: Verify the Build

```bash
# Docker
docker images | grep my-ee

# Podman
podman images | grep my-ee
```

The build process will:
1. Create a base image with Python, tar, and gzip
2. Extract and install HORCM binaries
3. Install Ansible Galaxy collections (hitachivantara.vspone_block)
4. Configure the execution environment with required Python packages
5. Set up a working container ready for AAP integration

## Using the Execution Environment

### With Ansible Automation Platform (AAP)

1. Push the image to your container registry:
   ```bash
   docker push quay.io/myregistry/hitachi-ee:1.0
   ```

2. In AAP, configure a new Execution Environment with the image URL

3. Use it in job templates for Hitachi storage automation tasks

### Standalone Usage

Run the container directly:

```bash
docker run -it my-ee:latest bash
```

From within the container, you can execute `raidcom` commands:

```bash
raidcom get resource -IH100
```

## Configuration

### HORCM Configuration Template

The `horcm100.conf.j2` file is a Jinja2 template for HORCM configuration:

```jinja2
HORCM_CMD
\.\IPCMD-{{ controller_ip }}-31001
```

This template can be rendered with your Hitachi storage controller IP address during playbook execution.

## Build Output

Ansible Builder generates:

- **Containerfile**: The Docker build file used to create the image
- **_build/**: Directory containing all artifacts prepared for building:
  - `bindep.txt` - Processed system dependencies
  - `requirements.txt` - Processed Python dependencies
  - `requirements.yml` - Galaxy collections manifest
  - `configs/` - Configuration files and HORCM archive
  - `scripts/` - Build scripts for container setup

## Troubleshooting

### HORCM.tgz not found
**Error**: `src: HORCM.tgz does not exist`

**Solution**: Ensure the `HORCM.tgz` file is in the repository root directory

### Build fails during HORCM installation
**Error**: `horcminstall.sh: No such file or directory`

**Solution**: Verify the HORCM.tgz archive structure contains the expected installation scripts

### Collection installation fails
**Error**: `Failed to download the collection hitachivantara.vspone_block`

**Solution**: 
- Check internet connectivity
- Verify the collection exists on Ansible Galaxy
- Set `ANSIBLE_GALAXY_DISABLE_GPG_VERIFY=1` if GPG verification fails (see Containerfile)

### Ansible Builder not found
**Error**: `command not found: ansible-builder`

**Solution**: Activate the virtual environment first, then verify the installation:
```bash
source .venv/bin/activate
ansible-builder --version
# If still missing:
pip install --upgrade ansible-builder
```

### exec format error (Apple Silicon)
**Error**: `exec /bin/sh: exec format error` or `image platform (linux/amd64) does not match the expected platform`

**Solution**: The HORCM binaries and the CentOS Stream 9 base image must be built for `linux/amd64`. Set the platform before building:
```bash
export DOCKER_DEFAULT_PLATFORM=linux/amd64
ansible-builder build -t my-ee:latest --container-runtime docker
```

### Docker Desktop not running
**Error**: `Cannot connect to the Docker daemon`

**Solution**: Launch Docker Desktop from your Applications folder and wait for it to report "running" in the menu bar before retrying.

### Podman machine not started
**Error**: `Error: Cannot connect to Podman. Please verify...`

**Solution**: Initialize and start the Podman machine:
```bash
podman machine init    # only needed once
podman machine start
```

### macOS tar adds extended attributes to HORCM.tgz
**Symptom**: Build fails with unexpected files or errors inside the container when extracting the tarball

**Solution**: Re-create the tarball with macOS extended attributes disabled:
```bash
COPYFILE_DISABLE=1 tar -cvzpf HORCM.tgz ./HORCM
```

### pip install fails due to externally managed environment
**Error**: `error: externally-managed-environment`

**Solution**: Use a virtual environment instead of installing into the system/Homebrew Python:
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install ansible-builder
```

## Documentation References

- [Ansible Builder Documentation](https://ansible-builder.readthedocs.io/)
- [Ansible Automation Platform Execution Environments](https://docs.ansible.com/automation-platform/latest/userguide/execution-environments.html)
- [Hitachi VSP One Block Collection](https://galaxy.ansible.com/hitachivantara/vspone_block)
- [Ansible Galaxy Collections](https://galaxy.ansible.com/)

## License

[Specify your license here]

## Support

For issues or questions about building and using this Execution Environment, please refer to:
- Ansible documentation
- Hitachi support resources
- Your organization's Ansible Automation Platform administrators
