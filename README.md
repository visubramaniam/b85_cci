# B85 CCI - Ansible Automation Platform Execution Environment

A custom Ansible Automation Platform (AAP) Execution Environment that bundles Hitachi Command Control Interface (CCI/HORCM) binaries to enable execution of `raidcom` commands against Hitachi storage arrays.

## Platform-Specific Build Instructions

Choose the guide for your operating system:

| Platform | Guide |
|---|---|
| **Linux** | [README.linux.md](README.linux.md) |
| **macOS** (Intel & Apple Silicon) | [README.macos.md](README.macos.md) |

## Overview

This repository builds a containerized Ansible Execution Environment based on CentOS Stream 9 that includes:

- **Ansible Core** and **Ansible Runner** for orchestration
- **HORCM (Hitachi RAID Manager)** binaries for storage array management
- **Hitachi VSP One Block collection** for Ansible playbooks
- System dependencies required for HORCM installation and operation

The resulting container can be used with Ansible Automation Platform (AAP) or standalone for executing storage automation tasks against Hitachi storage systems.

## Project Structure

```
.
├── README.md                          # This file (landing page)
├── README.linux.md                    # Linux build instructions
├── README.macos.md                    # macOS build instructions
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

## Documentation References

- [Ansible Builder Documentation](https://ansible-builder.readthedocs.io/)
- [Ansible Automation Platform Execution Environments](https://docs.ansible.com/automation-platform/latest/userguide/execution-environments.html)
- [Hitachi VSP One Block Collection](https://galaxy.ansible.com/hitachivantara/vspone_block)
- [Ansible Galaxy Collections](https://galaxy.ansible.com/)
