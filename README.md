# LUKS Unlocker Pro

![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/fernandoenzo/luks-unlocker-pro)
![GitHub](https://img.shields.io/github/license/fernandoenzo/luks-unlocker-pro)

A POSIX shell library for unlocking and mounting LUKS-encrypted devices during early boot (`initramfs`) on Debian-based Linux systems.

This project addresses use cases that are not easily handled by `cryptsetup` in conjunction with `crypttab`, such as unlocking the root partition using a detached LUKS header and/or a decryption key stored on external USB drives.

The library provides a set of functions that you can use to write your own boot script, giving you full control over which devices are unlocked, in what order, and with what parameters.

> **Warning:** Partitions unlocked using this program should **not** be added to `crypttab`, to avoid conflicts during boot. The recommended approach is to handle **all** encrypted partitions exclusively through `luks-unlocker-pro` and leave `crypttab` empty.

## Requirements

- A Debian-based distribution (Debian, Ubuntu, etc.). Functionality on other distributions is not guaranteed.
- The `cryptsetup` package, installed and enabled in the initramfs.
- The `initramfs-tools` package (standard on Debian-based systems).

## Installation

Copy the two files to their respective locations:

```sh
cp luks-unlocker-pro /etc/initramfs-tools/scripts/luks-unlocker-pro
cp custom_luks_decrypt /etc/initramfs-tools/scripts/local-top/custom_luks_decrypt
```

Make sure the boot script is executable:

```sh
chmod +x /etc/initramfs-tools/scripts/local-top/custom_luks_decrypt
```

Then rebuild the initramfs:

```sh
update-initramfs -u
```

> **Important:** You must run `update-initramfs -u` every time you modify either of these files.

## Files

| File | Installed to | Purpose |
|---|---|---|
| `luks-unlocker-pro` | `/etc/initramfs-tools/scripts/luks-unlocker-pro` | Function library (sourced by boot scripts) |
| `custom_luks_decrypt` | `/etc/initramfs-tools/scripts/local-top/custom_luks_decrypt` | Example boot script that uses the library |

## Functions

### `unlock_device(device, mapper_name, max_attempts, header, keyfile)`

Unlocks a LUKS-encrypted device. If the device is already unlocked, it returns immediately.

When no key file is provided (or when it is set to `"-"`), the function first tries any passwords previously entered during the current boot (stored in a temporary file), and then prompts the user interactively up to `max_attempts` times. Successfully entered passwords are stored for reuse with subsequent devices.

| Argument | Required | Default | Description |
|---|---|---|---|
| `device` | Yes | | Device path (e.g., `/dev/sda1` or `/dev/disk/by-uuid/...`) |
| `mapper_name` | Yes | | Name for the mapped device (e.g., `root`) |
| `max_attempts` | No | `3` | Maximum password attempts. Set to `0` to skip interactive prompt |
| `header` | No | | Path to a detached LUKS header file |
| `keyfile` | No | | Path to a key file. Use `"-"` for interactive password input |

### `mount_device(device, folder_name)`

Mounts a device under `/mnt`. If the device path starts with `/`, it is used as-is. Otherwise, it is treated as a device-mapper name and resolved to `/dev/mapper/<device>`.

| Argument | Required | Default | Description |
|---|---|---|---|
| `device` | Yes | | Absolute device path (e.g., `/dev/sda1`) or mapper name (e.g., `root`) |
| `folder_name` | No | basename of `device` | Folder name under `/mnt` |

```sh
mount_device "root"                # Mounts /dev/mapper/root at /mnt/root
mount_device "/dev/sda1" "debian"  # Mounts /dev/sda1 at /mnt/debian
```

### `unlock_and_mount_device(device, mapper_name, max_attempts, header, keyfile)`

Convenience function that calls `unlock_device` followed by `mount_device`. Takes the same arguments as `unlock_device`; the device is mounted under `/mnt/<mapper_name>`.

### `umount_and_close_device(mapper_name)`

Unmounts `/mnt/<mapper_name>` and closes the LUKS device `/dev/mapper/<mapper_name>`. Returns the exit code of the first operation that fails, or `0` if both succeed.

### `shred_passwords(iterations)`

Securely deletes the temporary file where passwords are stored, using `shred`. Should be called once all devices have been unlocked.

| Argument | Required | Default | Description |
|---|---|---|---|
| `iterations` | No | `10` | Number of overwrite passes |

### `success_or_shell(action, command)`

Executes a command. If it fails, it securely deletes stored passwords and drops the user into an interactive recovery shell (`panic`). The `action` argument controls what happens when the user exits the shell:

| Action | On shell exit | On tmpfile deletion |
|---|---|---|
| `continue` | Retries the failed command | Skips the command, returns `1` |
| `exit` | Retries the failed command | Aborts the script with `exit 1` |
| `rerun` | Re-executes the entire boot script | Aborts the script with `exit 1` |

> **Note:** Aborting the script will not prevent the system from attempting to continue booting.

## Example

The included `custom_luks_decrypt` file demonstrates a typical scenario: unlocking a root partition that uses a detached LUKS header and a key file, each stored on separate encrypted USB drives.

```sh
#!/bin/sh

PREREQ=""
prereqs() { echo "$PREREQ"; }
case "$1" in prereqs) prereqs; exit 0 ;; esac

# Load the luks-unlocker-pro library
. /scripts/luks-unlocker-pro

# 1. Unlock and mount the USB drive containing the LUKS header
success_or_shell "exit" unlock_and_mount_device /dev/disk/by-uuid/1111-1111 "header"

# 2. Unlock and mount the USB drive containing the key file
success_or_shell "exit" unlock_and_mount_device /dev/disk/by-uuid/2222-2222 "keyfile"

header_file="/mnt/header/header_file.luks"
key_file="/mnt/keyfile/key_file.luks"

# 3. Unlock the root partition using the header and key file (no mount needed;
#    the system will mount root later in the boot process)
success_or_shell "exit" unlock_device /dev/disk/by-uuid/3333-3333 "root" 0 "$header_file" "$key_file"

# 4. Securely delete stored passwords
shred_passwords

# 5. Unmount and close the USB drives (they are no longer needed)
umount_and_close_device "header";  close_header=$?
umount_and_close_device "keyfile"; close_keyfile=$?

# 6. If either close operation failed, drop to recovery shell
success_or_shell "exit" [ $((close_header | close_keyfile)) -eq 0 ]
```

To adapt this to your own setup, replace the UUIDs with your actual device UUIDs (found via `blkid`) and adjust the header/keyfile paths accordingly. If you only need a manual password (no key file or header), a simpler script would look like:

```sh
. /scripts/luks-unlocker-pro

success_or_shell "exit" unlock_and_mount_device /dev/disk/by-uuid/YOUR-UUID "root"
shred_passwords
```

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## License

![GitHub](https://img.shields.io/github/license/fernandoenzo/luks-unlocker-pro)

This program is licensed under the
[GNU General Public License v3 or later (GPLv3+)](https://choosealicense.com/licenses/gpl-3.0/)
