# nvidia

Provision NVIDIA GPU drivers

This role installs and configures a host for NVIDIA GPU drivers and the CUDA userspace binaries, including the NVIDIA System Management Interface (`nvidia-smi`). It sets up the necessary repositories (including specific ones for Red Hat distributions), installs the DKMS kernel module, blacklists the conflicting in-tree `nouveau` driver, and optionally installs the NVIDIA Container Toolkit so container runtimes can expose GPUs to workloads. See [Installing the NVIDIA driver on RHEL](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html) and the [NVIDIA Container Toolkit installation guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) for details.

The role will:
- Add and configure the NVIDIA CUDA and NVIDIA Container Toolkit repositories for the target operating system (with specific options for Red Hat).
- Install the DKMS build chain, including the `kernel-devel` and `kernel-headers` packages matching the *running* kernel.
- Install the NVIDIA driver and CUDA userspace packages, which provide `nvidia-smi`.
- Install the NVIDIA DKMS kernel module and reboot the host to activate it.
- Blacklist the `nouveau` driver and regenerate the initramfs so that the NVIDIA module can bind to the GPU on boot.
- Optionally install the NVIDIA Container Toolkit.

To confirm the driver is active, log into the host and run:

```bash
nvidia-smi
```

This should report the GPU model, driver version, and memory. If it fails, check that the kernel module built and that `nouveau` is not holding the device:

```bash
dkms status
lsmod | grep -E "nvidia|nouveau"
sudo journalctl -b | grep -i nvidia
```

The open-source kernel module flavour (`kmod-nvidia-open-dkms`, the default) requires a Turing or newer GPU — this includes the T4, A10G, A100, L4, L40S, and H100. For older Volta, Pascal, or Maxwell hardware, set `nvidia_kmod_package` to `kmod-nvidia-latest-dkms`.

Note that DKMS builds the module against the running kernel. A subsequent kernel upgrade that does not rebuild the module will leave the GPU unavailable after the next reboot.

To make GPUs schedulable by a container orchestrator, the NVIDIA Container Toolkit installed by this role must additionally be registered with the container runtime (for example via `nvidia-ctk runtime configure`), and a device plugin must advertise the resource to the scheduler. Those steps are deliberately outside the scope of this role.

# Requirements

- An NVIDIA GPU must be present on the target host.
- Compatible Linux distribution. The repository parameters in this role target Red Hat-based systems.
- Internet access from the target host to download packages and repository keys.
- Root or `sudo` privileges are required to manage package repositories, install packages, and configure system drivers.
- The role reboots the host to activate the kernel module.

# Dependencies

None.

# Parameters

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `nvidia_kmod_package` | `str` | `False` | `kmod-nvidia-open-dkms` | Kernel module package providing the NVIDIA driver. The open flavour requires a Turing or newer GPU. |
| `nvidia_container_toolkit_enabled` | `bool` | `False` | `True` | Whether to provision the NVIDIA Container Toolkit and its repository. |
| `nvidia_crb_repo_name` | `str` | `False` | | **(Red Hat Specific)** Name of the CodeReady Builder (CRB) repository to enable. |
| `nvidia_epel_rpm` | `str` | `False` | | **(Red Hat Specific)** URL or path to the EPEL RPM package to install. |
| `nvidia_python_packages` | `list` of `str` | `False` | | List of system Python packages to install. |
| `nvidia_build_packages` | `list` of `str` | `False` | | **(Red Hat Specific)** Packages required to build the DKMS kernel module, including the kernel headers matching the running kernel. |
| `nvidia_driver_packages` | `list` of `str` | `False` | | NVIDIA driver and CUDA userspace packages to install. |
| `nvidia_repo_name` | `str` | `False` | | **(Red Hat Specific)** Name (identifier) of the NVIDIA CUDA repository to configure. |
| `nvidia_repo_baseurl` | `str` | `False` | | **(Red Hat Specific)** Base URL of the NVIDIA CUDA repository. |
| `nvidia_repo_gpgkey` | `str` | `False` | | **(Red Hat Specific)** URL of the NVIDIA CUDA repository GPG key for package signature verification. |
| `nvidia_container_toolkit_packages` | `list` of `str` | `False` | | NVIDIA Container Toolkit packages to install. |
| `nvidia_container_toolkit_repo_name` | `str` | `False` | | **(Red Hat Specific)** Name (identifier) of the NVIDIA Container Toolkit repository to configure. |
| `nvidia_container_toolkit_repo_baseurl` | `str` | `False` | | **(Red Hat Specific)** Base URL of the NVIDIA Container Toolkit repository. |
| `nvidia_container_toolkit_repo_gpgkey` | `str` | `False` | | **(Red Hat Specific)** URL of the NVIDIA Container Toolkit repository GPG key for metadata signature verification. |

# Example Playbook

```yaml
- hosts: gpu_nodes
  become: true
  tasks:
    - name: Provision NVIDIA drivers with defaults
      ansible.builtin.import_role:
        name: cloudera.exe.nvidia
      # Installs nvidia-driver-cuda, kmod-nvidia-open-dkms and the
      # NVIDIA Container Toolkit, then reboots the host.

    - name: Provision NVIDIA drivers for pre-Turing GPUs, without the container toolkit
      ansible.builtin.import_role:
        name: cloudera.exe.nvidia
      vars:
        nvidia_kmod_package: "kmod-nvidia-latest-dkms"
        nvidia_container_toolkit_enabled: false

    - name: Provision NVIDIA drivers against an internal mirror (Red Hat)
      ansible.builtin.import_role:
        name: cloudera.exe.nvidia
      vars:
        nvidia_crb_repo_name: "codeready-builder-for-rhel-9-rhui-rpms"
        nvidia_repo_name: "cuda-rhel9"
        nvidia_repo_baseurl: "https://mirror.example.com/cuda/repos/rhel9/x86_64"
        nvidia_repo_gpgkey: "https://mirror.example.com/cuda/repos/rhel9/x86_64/D42D0685.pub"
```

# License

```
Copyright 2025 Cloudera, Inc.

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
```
