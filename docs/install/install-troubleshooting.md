# Install Troubleshooting

## Booting in Debug Mode

Modify the Boot Parameters (GRUB/Boot Menu)

This is often the most reliable way to get a shell or verbose output when direct TTY switching fails. You'll need to intercept the boot process.

1. Reboot the machine with the ISO.
2. At the GRUB (bootloader) menu: As soon as you see the initial boot menu, press an arrow key (up/down) to stop the automatic countdown.
3. Edit the boot entry: Select the installer's default boot entry (usually the first one). Press the `e` key to edit the boot parameters.
4. Locate the `linux` or `linuxefi` line: This line contains the kernel arguments. Add debug/shell parameters to force an emergency shell. Go to the end of the `linux` or `linuxefi` line and add the param (see choices below). This will drop you into an initramfs shell before the root filesystem is mounted. It's a very minimal environment but allows `ip a show`, `dmesg`, and looking at files in the initramfs.
5. Press `Ctrl+X` or `F10` to boot with the modified parameters.

The three main alternatives are:

**`rd.break`**

How it works: This parameter tells dracut (the initramfs environment) to interrupt the normal boot process before the real root filesystem is mounted and before systemd takes over. You are dropped into an initramfs shell where you can manually mount /sysroot and chroot into it to make changes. Since this is very early in the boot sequence, the environment is limited, but it lets you work on the system before anything else initializes.

When to use it: This is the method of choice for situations where you need to fix problems directly on the root filesystem, especially when you can't log in normally—such as resetting a forgotten root password or repairing critical files before systemd runs. It's lower-level than systemd.unit=emergency.target and gives you control before the system even mounts root.

**`systemd.unit=emergency.target`**

How it works: This parameter tells the systemd process to boot directly into a minimal shell. It will try to mount the root filesystem as read-only and start only the most essential services required for an emergency shell. This is often the preferred method for general system troubleshooting.

When to use it: This is a good choice for fixing issues that aren't related to the initial filesystem or the boot process itself, such as a corrupt /etc/fstab or a misconfigured service. It gives you a more complete environment than rd.break but still keeps things simple.

**`init=/bin/bash`**

How it works: This is the most direct and basic method. It tells the kernel to bypass all the normal boot processes and execute /bin/bash directly as the first process (PID 1). This gives you a shell with no services, no network, and the root filesystem mounted as read-only.

When to use it: Use this as a last resort when other methods fail. It's the most primitive and powerful method, as it gives you control before any other processes or services start. It's ideal for a corrupted boot process or severe filesystem issues where rd.break or emergency.target might fail to load. It also requires you to manually remount the root filesystem as read-write, just like you were doing with rd.break.

## Known Issues

* [Installing an OpenShift 4 cluster with the Agent-based Installer fail with "tls: failed to verify certificate: x509: certificate signed by unknown authority"](https://access.redhat.com/solutions/7070904). This usually happens when you are using some sort of web proxy and the certificate being presented for proxied connections was an intermediate certificate from the proxy. You need to add THAT root certificate and intermediaries to the installer to be able to trust the cert. 
