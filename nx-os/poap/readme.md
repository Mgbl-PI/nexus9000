# Nexus 9000 POAP Script (`poap.py`)

This repository contains a Python-based POAP (PowerOn Auto Provisioning) workflow for Cisco Nexus platforms. The script automates first-boot provisioning, including config download, image handling, optional multi-step upgrades, and optional post-provision package installation (licenses/RPMs/certificates).

## What this script does

`poap.py` performs the following high-level flow:

1. Validates and applies default `options`.
2. Initializes logging and syslog prefixing.
3. Selects provisioning mode (`serial_number`, `mac`, `hostname`, `location`, `personality`, `raw`).
4. Verifies bootflash free space and prepares destination directories.
5. Handles optional bundle workflow (on supported target images).
6. Copies/validates configuration and splits it when required for older image behavior.
7. Copies optional user apps/scripts.
8. Optionally parses per-device YAML and schedules license/RPM/certificate operations.
9. Copies system (and kickstart if needed), installs image, and schedules config replay.

---

## Editable configuration

User-editable settings are at the top of `poap.py` in the `options` dictionary and global flags.

### Minimal required options (remote POAP)

- `username`
- `password`
- `hostname`
- `transfer_protocol`
- `mode`
- `target_system_image` (not required in `personality` mode)

### Important defaults and options

- `mode` (default: `serial_number`)
- `transfer_protocol` (default: `scp`)
- `config_path` (default: `/var/lib/tftpboot/`)
- `target_image_path` (default: `/var/lib/tftpboot/`)
- `target_system_image`, `target_kickstart_image`
- `destination_path` (default: `/bootflash/`)
- `required_space` (KB)
- `disable_md5` (default: `False`)
- `https_ignore_certificate` (for HTTPS copy)
- `compact_image` (compact SCP image copy)
- `skip_multi_level`, `midway_system_image`, `midway_kickstart_image`
- `bundle_name` (bundle mode trigger when non-empty)
- `install_path` (enables YAML-driven License/RPM/Certificate flow)
- `user_app_path` (for `download_scripts_and_agents()` content)
- `usb_slot`, `source_config_file`, `vrf`

### Global behavior flags

- `global_use_kstack = False`
- `global_upgrade_bios = False`
- `global_copy_image = True`

---

### Config file naming based on the supported modes

- **serial_number**: `conf.<POAP_SERIAL>`
- **mac**: `conf_<POAP_MAC>.cfg` (USB mode includes router/mgmt MAC fallback behavior)
- **hostname**: `conf_<POAP_HOST_NAME>.cfg`
- **location**: `conf_<cdp_switch>_<cdp_interface>.cfg`
- **raw**: uses `source_config_file` as-is
- **personality**: downloads personality tarball and derives target image from tar contents

---

## Integrity checks

By default, MD5 validation is enabled for downloaded artifacts (config, images, tarballs, YAML recipe where applicable).

- If `disable_md5` is `False`, the script attempts to copy corresponding `.md5` files and verify checksums.
- If checksums fail, provisioning aborts.

---

## Optional YAML-driven install flow (`install_path`)

If `install_path` is set and mode is not `personality`, the script tries to load a per-device YAML file:

- `<install_path>/<serial>/<serial>.yaml` (fallback to `.yml`)

This YAML supports:

- `Version` (must be `1`)
- `Target_image` (optional override of common target image)
- `License` (list of `.lic` files)
- `RPM` (list of `.rpm` files)
- `Certificate` (list of cert/key files copied to `bootflash/poap_files`)
- `Trustpoint` (map of trustpoint name -> `{pkcs12_file: passphrase}`)

The script validates extensions, copies artifacts, and appends install commands to scheduled config where needed.

### YAML file location and behavior

When `install_path` is configured, POAP expects a per-device YAML recipe at:

- `<install_path>/<serial-number>/<serial-number>.yaml`
- Fallback: `<install_path>/<serial-number>/<serial-number>.yml`

Example:

- If `install_path` is `/tftpboot/`
- Expected file for a device is `/tftpboot/<serial-number>/<serial-number>.yaml`

If the file is present and valid, POAP uses it to schedule per-device package and certificate operations.
If the file is missing, POAP logs that condition and continues with legacy POAP workflow.

### YAML keys and expectations

- `Version`
  - Mandatory key
  - Must be set to `1` for this release
- `License`
  - List of `.lic` files
  - Paths must be relative to `install_path`
- `RPM`
  - List of `.rpm` files
  - Paths must be relative to `install_path`
- `Certificate`
  - List of certificate/public-key files without trustpoint import
  - These are copied to `bootflash/poap_files/`
- `Trustpoint`
  - Trustpoint map with PKCS12 file and passphrase
  - Supported PKCS12 extensions are `.pfx` and `.p12`
- `Target_image`
  - Optional per-device override of common `target_system_image`
  - Must be image filename only (no relative path support)
  - The image must be available under `target_image_path`

### Sample YAML file

Sample filename: `XYZ12345.yaml`

```yaml
Version: 1
Certificate:
  - ssh_key1.pub
  - XYZ12345/nxapi_server_key.pem
  - XYZ12345/nxapi_server_cert.pem
License:
  - XYZ12345/XYZ12345_1.lic
  - XYZ12345_2.lic
RPM:
  - POAP_TPARTY_AND_PATCH_RPMS/chef-12.19.33-1.nexus7.x86_64.rpm
  - mtx-openconfig-vlan-1.0.0.206-9.3.5.lib32_n9000.rpm
  - POAP_TPARTY_AND_PATCH_RPMS/nxos.POAP_SMU_BGP_RELOAD-n9k_ALL-1.0.0-9.3.5.lib32_n9000.rpm
Target_image: nxos.9.3.5.bin
Trustpoint:
  Z1_TP:
    POAP_TP_FILES/XYZ12345/Z1_TP.pfx: passphrase1
  Z2_TP:
    POAP_TP_FILES/XYZ12345/Z2_TP.p12: passphrase2
```

---

## Bundle mode

When `bundle_name` is non-empty, the script attempts bundle workflow if supported:

- Support check is based on target image parsing (script logic expects bundle support from `>= 10.7(1)` style target image naming).
- Tries `<bundle>.tar`, then `<bundle>.tgz` from `config_path`.
- Extracts into `/bootflash/poap_pending/`.
- Requires `poap.cfg` in bundle and checks for `feature openconfig` before using it.

If bundle fails or unsupported, script falls back to normal config copy workflow.

---

## Multi-step upgrade flow (`multi_step_install`)

### Why this exists

Some upgrade paths cannot jump directly from the current image to the final target image (taking into account the ISSU matrix). In those cases, the `midway_system_image` needs to be populated by the user. 


### When `multi_step_install` becomes active

- If `midway_system_image` are configured and the switch is not already booted on that midway image.

### What happens in practice

1. **First POAP run (staged image run):**
  - Script focuses on image copy/install for the midway image.
  - It skips normal config/bundle/user-app/YAML install tasks in this pass.
  - It sets boot variables and triggers write erase/reboot behavior so POAP runs again.

2. **Second POAP run (final provisioning run):**
  - Script continues normal workflow (config copy, optional bundle, optional YAML-driven installs, final image actions).
  - It applies scheduled configuration as part of the normal completion path.

### Operator controls

- `midway_system_image` allow explicit control of the intermediate hop.

---

## USB behavior

When `POAP_PHASE=USB`:

- Remote credentials are not required.
- Copy source paths use `/usbslot<usb_slot>/...`.
- Personality mode is not supported over USB.

---

## Logs and cleanup

- Runtime logs are written under `/bootflash/*poap*script.log`.
- Script keeps the most recent 4 POAP logs.
- On abort, cleanup and rollback paths attempt to remove temporary files and revert scheduled RPM/license/certificate changes where possible.

---

## Updating script integrity hash

The script header contains an internal MD5 comment (`#md5sum="..."`).
If the script is modified, regenerate this value using the command in the script header comments.

The following command is for bash-shell. 

```bash
f=poap_nexus_script.py ; cat $f | sed '/^#md5sum/d' > $f.md5 ; sed -i \
"s/^#md5sum=.*/#md5sum=\"$(md5sum $f.md5 | sed 's/ .*//')\"/" $f
```

---

## Typical customization checklist

1. Edit the top-level `options` dictionary for your environment.
2. Set `mode` to match your config naming strategy.
3. Verify `config_path`, `target_image_path`, and protocol credentials.
4. Decide whether to keep MD5 verification enabled.
5. (Optional) Enable `install_path` and create per-device YAML recipes.
6. (Optional) Customize `download_scripts_and_agents()` for user agents/scripts.

---

## Notes

- This script is intended to run in the NX-OS POAP environment where POAP-specific environment variables and Cisco CLI bindings are available.
- It is not a generic standalone Python provisioning tool.
