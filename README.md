# LUKS Unlocker Pro

![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/fernandoenzo/luks-unlocker-pro)
![GitHub](https://img.shields.io/github/license/fernandoenzo/luks-unlocker-pro)

LUKS Unlocker Pro is a project focused on unlocking and mounting LUKS encrypted devices in a customized and flexible way in the `initramfs`. This program will work exclusively on Debian
distributions and probably its derivatives. Its functionality is not guaranteed on other distributions.

The goal of this project is to address specific use cases that are not easily handled by `cryptsetup` in conjunction with `crypttab`. For example, if you want to unlock the `root` partition using a
detached LUKS header and/or a decryption key stored on one or more USB drives.

LUKS Unlocker Pro includes a set of powerful functions that allow you to create your own script to run at system startup
and unlock and mount the LUKS devices you want with the parameters you choose.

Warning: Please notice that if you unlock partitions using this program, they should not be added to the `crypttab` file to avoid conflicts, especially if we are talking about the `root`
partition.

My personal recommendation is that you use `luks-unlocker-pro` exclusively for partitions that you need to boot in the `initramfs` in a more personalized way, leaving `crypttab` for everything
else. Or you can just leave empty the `crypttab` and use exclusively `luks-unlocker-pro`.

## Files

- `/etc/initramfs-tools/scripts/luks-unlocker-pro`: This file contains a set of functions for unlocking and mounting LUKS encrypted devices.
- `/etc/initramfs-tools/scripts/local-top/custom_luks_decrypt`: This file is an example of a custom script that utilizes the functions defined in `luks-unlocker-pro` to unlock and mount specific
  devices during system boot.

## Included Functions

The `luks-unlocker-pro` script contains several functions for unlocking and mounting LUKS encrypted devices. These functions include:

- `exec_silently(command)`: Executes the given command silently, suppressing any output. This function is useful for running commands that produce unnecessary output or when you want to keep the
  output clean.
- `print_text(text)`: Prints the given text using Plymouth if available, otherwise falls back to echo command.
- `is_device_mounted(mount_point)`: Checks if a given mount point is already in use by a mounted device.
- `unlock_device(device, mapper_name, max_attempts, header, keyfile)`: Unlocks a LUKS encrypted device using `cryptsetup` with provided options. It attempts to unlock the specified device using
  the provided information, including previously entered passwords for unlocking other devices.
- `mount_device(device, is_mapped_device, folder_name)`: This function mounts a device to a mount point based on the folder name, considering whether the device is a mapped device or not. It
  checks if the device is already mounted, determines the file system type, and mounts the device accordingly, outputting success or error messages. This simplifies the process of mounting devices in
  Linux systems by handling different device types and automating the process.
- `unlock_and_mount_device(device, mapper_name, max_attempts, header, keyfile)`: This function combines the functionality of `unlock_device` and `mount_device` to unlock and mount a LUKS device in one
  step.
- `umount_and_close_device(mapper_name)`: Unmounts and closes a LUKS encrypted device.
- `shred_passwords(iterations)`: This function securely deletes the temporary password file used by `unlock_device`.
- `success_or_shell(action, command)`: Executes a command and, if it fails, provides an interactive shell to troubleshoot the issue and perform a specific action based on the action argument. This
  function is useful for debugging and real-time problem-solving during script execution.

## Updating the `initramfs`

After adding the files to your system or making any changes, it is important to update the initramfs to ensure that the changes take effect during system startup. To do this, run the
command `update-initramfs -u`.

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## License

![GitHub](https://img.shields.io/github/license/fernandoenzo/luks-unlocker-pro)

This program is licensed under the
[GNU General Public License v3 or later (GPLv3+)](https://choosealicense.com/licenses/gpl-3.0/)
