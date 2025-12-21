# B85 CCI - Ansible Automation Platform Execution Environment

A custom Ansible Automation Platform (AAP) Execution Environment that bundles Hitachi Command Control Interface (CCI/HORCM) binaries to enable execution of `raidcom` commands against Hitachi storage arrays.

## Overview

This repository builds a containerized Ansible Execution Environment based on CentOS Stream 9 that includes:

- **Ansible Core** and **Ansible Runner** for orchestration
- **HORCM (Hitachi RAID Manager)** binaries for storage array management
- **Hitachi VSP One Block collection** for Ansible playbooks
- System dependencies required for HORCM installation and operation

The resulting container can be used with Ansible Automation Platform (AAP) or standalone for executing storage automation tasks against Hitachi storage systems.

## Prerequisites

Before building this Execution Environment, ensure you have:

1. **Ansible Builder** installed:
   ```bash
   pip install ansible-builder
   ```

2. **Docker or Podman** installed and running (required by Ansible Builder)

3. **HORCM.tgz** archive:
   - Obtain the HORCM binaries from Hitachi
   - Run the commands below to create HORCM.tgz
   - cat RMHORC | cpio -ivdum
   - chmod -R 777 ./HORCM
   - chown -R root:sys ./HORCM
   - tar -cvzpf HORCM.tgz ./HORCM
   - Place the `HORCM.tgz` file in the repository root directory
   - This file is referenced in `execution_environment.yml` and will be included in the container during build

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

## Building the Execution Environment

### Step 1: Add HORCM Binaries

Ensure the `HORCM.tgz` file is in the repository root:

```bash
ls -la HORCM.tgz
```

### Step 2: Build the Container Image

Run the ansible-builder command:

```bash
ansible-builder build -t my-ee:latest
```

**Options:**
- `-t, --tag` - Tag for the built image (e.g., `localhost/my-ee:1.0`)
- `--verbosity` - Increase verbosity (e.g., `-vv`)
- `--container-runtime` - Use `docker` or `podman` (auto-detected by default)

**Example with full options:**

```bash
ansible-builder build \
  -t quay.io/myregistry/hitachi-ee:1.0 \
  -v 3 \
  --container-runtime docker
```

### Step 3: Verify the Build

```bash
docker images | grep my-ee
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

**Solution**: Install ansible-builder:
```bash
pip install --upgrade ansible-builder
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
