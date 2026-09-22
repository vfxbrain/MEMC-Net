# Recovering a PC after a session disabled the USB controller

## Before you touch anything: get your disk encryption key

If the drive uses BitLocker (Windows) or LUKS-with-TPM (Linux), changing
firmware settings or clearing CMOS will make the machine demand a recovery key
on the next boot. Get it first, from another device:

- BitLocker: https://aka.ms/myrecoverykey (sign in with the same Microsoft account)
- Or the printout / USB key you saved when encryption was turned on

If you have no key and the drive is encrypted, do NOT clear CMOS. Use the
software route (Step 3) only.

---

## Step 1 — Work out which layer is broken

Power on and immediately tap the setup key (Del, F2, F10, or F12, depending on
the board).

- Keyboard works in the firmware setup screen -> the firmware is fine, the
  problem is inside the operating system. Go to Step 3.
- Keyboard is dead even there, or the machine never reaches a screen -> the
  problem is at the firmware level. Go to Step 2.

Before concluding the keyboard is dead, try these. They cost nothing:

- Plug the keyboard directly into a rear USB 2.0 port on the motherboard.
  Not a hub, not a monitor port, not a front-panel port.
- Try every rear port. Boards often disable one controller and not the others.
- On a laptop, the built-in keyboard and trackpad usually keep working because
  they hang off the internal controller, not the external USB one. If they
  respond, you can skip straight to Step 3 and do everything from inside the OS.
- A PS/2 keyboard, if the board has the purple port, bypasses USB entirely.

## Step 2 — Reset the firmware

This restores every firmware setting, including any USB controller, XHCI
hand-off, or legacy USB support that was switched off. It does not touch your
files.

If you can reach the setup screen: press the "load optimized defaults" key
(usually F9), confirm, then save and exit (usually F10).

If you cannot reach the setup screen, clear CMOS physically:

1. Shut down, switch off the power supply, pull the wall plug.
2. Hold the case power button for 15 seconds to drain residual charge.
3. Either move the CLR_CMOS / JBAT1 jumper to the second position for 10
   seconds and move it back, or lift out the coin cell battery for 2 minutes
   and reseat it. The motherboard manual names the exact jumper.
4. Reconnect and boot.

Many recent boards have a labelled "Clear CMOS" button on the rear I/O panel,
which does the same thing in one press.

After the reset, go back into setup and confirm these are enabled if the board
exposes them: USB Controller, XHCI Hand-off, Legacy USB Support, USB Keyboard
Support.

## Step 3 — Undo the change inside the operating system

### Windows

The fastest fix is System Restore, which rolls the driver and registry state
back to before the session without touching your documents.

1. Force the recovery environment: power on, and as soon as the Windows logo
   appears hold the power button until it cuts off. Do that three times. The
   fourth boot lands in Automatic Repair.
2. Advanced options -> Troubleshoot -> Advanced options -> System Restore.
3. Pick a restore point dated before the session. Let it finish and reboot.

If there is no usable restore point, use the command prompt in the same
recovery menu and repair the service entries directly. Check the drive letter
first, because in recovery Windows is often mounted as D: rather than C:.

    diskpart
    list volume
    exit

Then, substituting the real letter for C:

    reg load HKLM\OFF C:\Windows\System32\config\SYSTEM
    reg query HKLM\OFF\ControlSet001\Services\USBXHCI /v Start

A Start value of 4 means disabled. Repair the whole USB stack:

    reg add HKLM\OFF\ControlSet001\Services\USBXHCI /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\USBHUB3 /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\usbhub  /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\usbehci /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\HidUsb  /v Start /t REG_DWORD /d 3 /f
    reg add HKLM\OFF\ControlSet001\Services\kbdhid  /v Start /t REG_DWORD /d 3 /f
    reg add HKLM\OFF\ControlSet001\Services\mouhid  /v Start /t REG_DWORD /d 3 /f
    reg add HKLM\OFF\ControlSet001\Services\USBSTOR /v Start /t REG_DWORD /d 3 /f
    reg unload HKLM\OFF

If the machine has more than one control set, repeat the block for
ControlSet002. Also check for a blanket storage policy and delete it if present:

    reg load HKLM\OFFSW C:\Windows\System32\config\SOFTWARE
    reg delete "HKLM\OFFSW\Policies\Microsoft\Windows\RemovableStorageDevices" /f
    reg unload HKLM\OFFSW

Reboot.

Two other things a session might have done, both fixed from a working desktop
in an elevated PowerShell:

    Get-PnpDevice -Class USB | Where-Object Status -ne OK
    Get-PnpDevice -Class USB | Where-Object Status -eq Error |
        Enable-PnpDevice -Confirm:$false

### Linux

At the boot menu, highlight the entry and press `e` to edit it. Look at the
`linux` line for anything added to the kernel command line, such as
`nousb`, `usbcore.authorized_default=0`, `module_blacklist=xhci_hcd`, or
`xhci_hcd.blacklist=yes`. Delete just that text and press Ctrl-X to boot once
without it. That confirms the cause without changing anything on disk.

Once booted, or from a live USB with the root filesystem mounted at /mnt, make
it permanent by checking these four places:

    grep -rn 'xhci\|usbcore\|usb' /etc/modprobe.d/ /usr/lib/modprobe.d/
    grep -n GRUB_CMDLINE /etc/default/grub
    grep -rn 'SUBSYSTEM=="usb"' /etc/udev/rules.d/
    grep -rn 'usb' /etc/modules-load.d/

Remove any `blacklist xhci_hcd`, `install xhci_hcd /bin/true`, or udev rule
setting `ATTR{authorized}="0"`. Then rebuild both the boot config and the
initramfs, because a blacklist baked into the initramfs survives editing the
file alone:

    sudo update-grub          # or: grub2-mkconfig -o /boot/grub2/grub.cfg
    sudo update-initramfs -u  # or: dracut -f

If the machine will not boot at all, a live USB is the way in. Mount the root
partition, make the edits under /mnt, then chroot to rebuild:

    sudo mount /dev/nvme0n1p2 /mnt
    sudo mount /dev/nvme0n1p1 /mnt/boot/efi
    for d in dev proc sys run; do sudo mount --rbind /$d /mnt/$d; done
    sudo chroot /mnt

## Step 4 — Read the transcript to confirm what actually changed

The session log is on the same M.2 drive. Boot a live USB, or pull the drive
into another machine in an enclosure, then read it. Guessing is optional once
you have this.

Windows paths:

    C:\Users\<you>\.claude\projects\<encoded-project-path>\*.jsonl
    C:\Users\<you>\.claude\history.jsonl
    C:\Users\<you>\.claude.json
    C:\Users\<you>\.claude\shell-snapshots\

Linux and macOS paths:

    ~/.claude/projects/<encoded-project-path>/*.jsonl
    ~/.claude/history.jsonl

Search them for the culprit:

    grep -ril -e xhci -e usbcore -e USBSTOR -e Disable-PnpDevice \
      -e blacklist -e 'reg add' -e nousb ~/.claude/projects/

The `.jsonl` files are one JSON object per line. `jq -r 'select(.type=="user"
or .type=="assistant")'` makes them readable, or just open them in a text
editor and read the command strings.

Also worth checking, for edits made outside the session log:

    Windows: Get-WinEvent -FilterHashtable @{LogName='System'; Id=7040,7036}
    Linux:   journalctl -b -1 -p err ; ls -lt /etc/modprobe.d/ /etc/udev/rules.d/

## Order to try things

1. Different USB port, rear panel, USB 2.0.
2. Firmware setup -> load optimized defaults.
3. Clear CMOS, only after securing the encryption recovery key.
4. Windows System Restore, or Linux one-shot kernel command line edit.
5. Offline registry or config repair.
6. Read the transcript and undo precisely what it did.

Steps 1 through 4 risk nothing. Step 5 changes system files, so note the
original values before you overwrite them.
