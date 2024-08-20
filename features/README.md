# Features

## General
Each folder represents a usable Garden Linux `feature` that can be added to a final Garden Linux artifact. This allows you to build Garden Linux for different cloud platforms (e.g. `azure`, `gcp`, `container` etc.) with a different set of features like `CIS`, `read_only`, `firewall` etc. Currently, the following feature types are available:

| Feature Type | Feature Name |
|---|---|
| Platform | `ali`, `aws`, `azure`, `gcp`, `kvm`, `metal`, ... |
| Feature | `firewall`, `gardener`, `ssh`, ... |
| Modifier | `_slim`, `_readonly`, `_pxe`, `_iso`, ... |
| Element | `cis`, `fedramp`, ... |

*Keep in mind that `not all features` may be combined together. However, features may in-/exclude other features or block the build process by given exclusive/incompatible feature combinations.*

## Building a custom set of features and modifiers

Garden Linux utilizes the [gardenlinux/builder](https://github.com/gardenlinux/builder) to create customized Linux distributions. The `gardenlinux/gardenlinux` repository is maintained by the Garden Linux team, highlighting specialized "features" that are also available for other projects.

To initiate a build, navigate to the root directory of the `gardenlinux/gardenlinux` repository and use the command:

```bash
./build ${platform}-${feature}_${modifier}
```

Where:

- `${platform}` denotes the desired platform (e.g., kvm, metal, aws).
- `${feature}` represents a specific feature from the `features/` folder.
- `${modifier}` is an optional modifier from the `features/` folder, prefixed with an underscore "_".

You can combine multiple platforms, features, and modifiers as needed.

## Official Combinations

Garden Linux images are constructed during the [nightly GitHub action](https://github.com/gardenlinux/gardenlinux/blob/main/.github/workflows/nightly.yml). The following table showcases the flavors that are built and tested with each nightly run.


| Platform | feature/modifier combinations |
|----------|--------------------------------------------|
| KVM      | `kvm-gardener_prod`                        |
|          | `kvm_secureboot-gardener_prod`             |
|          | `kvm_secureboot_readonly-gardener_prod`    |
|          | `kvm_secureboot_readonly_persistence-gardener_prod` |
| Metal    | `metal-gardener_prod`                      |
|          | `metal_secureboot-gardener_prod`           |
|          | `metal_secureboot_readonly-gardener_prod`  |
|          | `metal_secureboot_readonly_persistence-gardener_prod` |
|          | `metal_pxe-gardener_prod`                  |
|          | `metal-vhost-gardener_prod`                |
| GCP      | `gcp-gardener_prod`                        |
| AWS      | `aws-gardener_prod`                        |
|          | `aws_secureboot-gardener_prod`             |
|          | `aws_secureboot_readonly-gardener_prod`    |
|          | `aws_secureboot_readonly_persistence-gardener_prod` |
| Azure    | `azure-gardener_prod`                      |
| Ali      | `ali-gardener_prod`                        |
| OpenStack| `openstack-gardener_prod`                  |
| VMware   | `vmware-gardener_prod`                     |
| Firecracker | `firecracker-gardener_prod`             |



## Summary of Changes for Gardenlinux OS Images for CAPI and Gardener-Extension

This documentation outlines the modifications made to support two specific flavors of the Gardenlinux OS images: one for **ClusterAPI** and another for the **Gardener Extension Provider for metalAPI**. Below, we detail the changes and additions made for each flavor.

### Gardener Extension Provider Flavor

#### Purpose
To ensure the correct mounting of `/var/lib/containerd` under the Gardener Extension provider to prevent issues with overlay filesystems on containerd.

#### Key Changes
- **Script Addition**: A script was added at `features/gardener/file.include/gardenlinux.dracut.end` to create and mount an ext4 filesystem dedicated to `/var/lib/containerd`.

    ```bash
    #!/usr/bin/env bash
    /sysroot/usr/bin/dd if=/dev/zero of=/sysroot/var/lib/containerd_storage.img bs=1M count=100000
    /usr/sbin/mkfs.ext4 /sysroot/var/lib/containerd_storage.img
    mkdir -p /sysroot/var/lib/containerd
    mount -o loop /sysroot/var/lib/containerd_storage.img /sysroot/var/lib/containerd
    ```
- Note: the changes related to the `_ignition` of adding ignitioV2 should be reverted for this flavor.

- **Build Command**: `./build metal-gardener_prod_pxe`

### ClusterAPI Flavor

#### Purpose
To adapt the OS image to support only IgnitionV2 under ClusterAPI and specific containerd configurations.

#### Key Changes
- **Ignition Integration**: The Ignition 2.18.0 package was included to handle configuration needs specific to ClusterAPI.

- **Containerd Configuration**: Adjustments were made to the containerd configuration to utilize the native snapshotter and integrate seamlessly with systemd cgroups.

    - **Configurations Added**:
        - Updated `features/_ignite/pkg.include` to include `dracut-network`, `curl`, and `gettext-base`.
        - Implemented an Ignition setup script at `features/_ignite/exec.config`:

        ```bash
        DEBIAN_FRONTEND=noninteractive dpkg -i /opt/*.deb
        apt-mark hold ignition
        rm -rf /opt/*.deb
        ```

        - New `containerd` configuration set in `features/khost/file.include/etc/containerd/config.toml`:

        ```toml
        version = 2

        [plugins]
          [plugins."io.containerd.grpc.v1.cri"]
            [plugins."io.containerd.grpc.v1.cri".containerd]
              snapshotter = "native"
              [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
                [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
                  runtime_type = "io.containerd.runc.v2"
                  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                    SystemdCgroup = true
        ```

- **New Feature Set `capi`**:
    - A new feature set for ClusterAPI was introduced, defined under `features/capi` with the following configuration:

    ```yaml
    description: 'cluster-api'
    type: platform
    features:
      include:
        - server
        - khost
        - metal
        - _pxe
      exclude:
        - _selinux
        - firewall
    ```

- **Build Command**: `./build capi`

### Image Releases
The tailored OS images are now available at the following locations:
- **Gardener Metal**: `ghcr.io/ironcore-dev/os-images/gardenlinux:1443.10-gardener-metal`
- **ClusterAPI**: `ghcr.io/ironcore-dev/os-images/gardenlinux:1443.10-capi`



