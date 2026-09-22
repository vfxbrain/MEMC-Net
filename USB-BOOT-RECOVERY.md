# Windows PC will not boot, debug display reads 15

## What code 15 means

The two-digit display on the motherboard is a POST code, called Q-Code on ASUS,
Dr. Debug on ASRock, and just the debug LED on Gigabyte and high-end MSI boards.
Almost every board uses the AMI Aptio code table, where 15 is:

    0x15 - pre-memory North Bridge initialization started

That is very early. The firmware has started the CPU and is about to train
memory. Windows has not loaded. No driver, no registry key, and no disabled
device has run yet.

So the practical conclusion is this: if the machine is genuinely stuck at 15,
the cause is not a USB controller that got switched off inside Windows.
Software running in Windows cannot normally stall a pre-memory POST. The only
routes by which it could are a firmware flash, a write to UEFI NVRAM
variables, or coincidence, meaning a hardware fault that surfaced around the
same time as the session.

Everything about System Restore and registry repair is therefore on hold. It
only becomes relevant once the board completes POST.

## First, make sure it is actually stuck

Boards pass through 15 on every normal boot, so a glimpse of it means nothing.

- Watch for a full five minutes without touching anything. Memory training on
  DDR5 and on AMD AM5 can take minutes, especially the first boot after a CMOS
  clear, and the display can sit on a pre-memory code the whole time.
- Note whether the number ever changes. A code that cycles or advances and then
  loops is a different problem from one frozen on 15.

## The free checks, in order

1. **Unplug every USB device.** Keyboard, mouse, hub, dock, headset, webcam,
   phone, external drive, everything, front panel and rear. A shorted port or a
   failing device really can stall POST, and this is the one place where your
   USB suspicion and a hang at 15 could actually meet. Boot with nothing
   attached and see whether the code moves past 15.

2. **Check the CPU power cable.** The 8-pin EPS connector at the top left of the
   board, not just the 24-pin. A partially seated EPS connector is a classic
   cause of a hang in the low POST codes. Unplug and reseat both until they
   click.

3. **Reseat the memory.** Power off and unplug first. Remove every stick. Boot
   with a single stick in the slot the manual names for one-module operation,
   usually the second slot out from the CPU, labelled A2 or DIMM_A2. If it
   POSTs, add sticks back one at a time. If it does not, try that same stick in
   each other slot, then try each other stick.

4. **Clear CMOS.** This wipes any stored memory overclock, XMP or EXPO profile
   that the board can no longer train. Unplug from the wall, hold the case power
   button fifteen seconds, then either move the CLR_CMOS jumper for ten seconds
   or pull the coin cell for two minutes. Some boards have a Clear CMOS button
   on the rear panel instead. If it boots afterwards, leave XMP and EXPO off
   until you are sure it is stable.

5. **Strip it down.** Disconnect all drives and every PCIe card except the
   graphics card. Fewer devices, fewer things to hang on.

6. **Reseat the CPU and inspect for bent pins.** On AM4 the pins are on the
   chip, on AM5 and Intel LGA they are in the socket. Look across the socket at
   a low angle under bright light. Reseat the cooler with even pressure, since
   an overtightened cooler can flex the socket.

7. **BIOS Flashback**, if the board has it. A button on the rear panel, often
   labelled BIOS Flashback or Q-Flash Plus, with a dedicated USB port beside it.
   It reflashes firmware from a FAT32 stick with no CPU, memory or graphics card
   needed, and it is the fix if firmware itself was corrupted. The file has to
   be renamed exactly as the board manual specifies.

## If it gets past POST and into Windows

Only then does the USB question matter. Recover in this order.

**System Restore.** Power on, and as soon as the Windows logo appears hold the
power button until it cuts off. Do that three times. The fourth boot lands in
Automatic Repair. Go to Troubleshoot, Advanced options, System Restore, and pick
a restore point dated before the session. It rolls back drivers and registry
state without touching your documents.

**Offline registry repair**, if no restore point exists. From the command prompt
in that same recovery menu, first find the real drive letter, since Windows is
often mounted as D: there.

    diskpart
    list volume
    exit

Then, substituting the letter you found:

    reg load HKLM\OFF C:\Windows\System32\config\SYSTEM
    reg query HKLM\OFF\ControlSet001\Services\USBXHCI /v Start

A Start value of 4 means disabled. Repair the stack:

    reg add HKLM\OFF\ControlSet001\Services\USBXHCI /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\USBHUB3 /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\usbhub  /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\usbehci /v Start /t REG_DWORD /d 0 /f
    reg add HKLM\OFF\ControlSet001\Services\HidUsb  /v Start /t REG_DWORD /d 3 /f
    reg add HKLM\OFF\ControlSet001\Services\kbdhid  /v Start /t REG_DWORD /d 3 /f
    reg add HKLM\OFF\ControlSet001\Services\mouhid  /v Start /t REG_DWORD /d 3 /f
    reg add HKLM\OFF\ControlSet001\Services\USBSTOR /v Start /t REG_DWORD /d 3 /f
    reg unload HKLM\OFF

Repeat for ControlSet002 if the machine has one. Also clear a blanket storage
policy if one was added:

    reg load HKLM\OFFSW C:\Windows\System32\config\SOFTWARE
    reg delete "HKLM\OFFSW\Policies\Microsoft\Windows\RemovableStorageDevices" /f
    reg unload HKLM\OFFSW

**From a working desktop**, in an elevated PowerShell:

    Get-PnpDevice -Class USB | Where-Object Status -ne OK
    Get-PnpDevice -Class USB | Where-Object Status -eq Error |
        Enable-PnpDevice -Confirm:$false

## Before you clear CMOS, get your BitLocker key

If the drive is encrypted, resetting firmware makes the next boot demand a
recovery key. Fetch it from your phone or another PC at
https://aka.ms/myrecoverykey, signed in with the same Microsoft account. If you
cannot produce the key, do not clear CMOS.

## Reading the session transcript

The log is on the same M.2. Put the drive in an enclosure on another PC, or boot
a Windows install USB and use the command prompt. Look in:

    C:\Users\<you>\.claude\projects\<encoded-project-path>\*.jsonl
    C:\Users\<you>\.claude\history.jsonl
    C:\Users\<you>\.claude.json
    C:\Users\<you>\.claude\shell-snapshots\

Search them for what was actually run. From PowerShell:

    Select-String -Path "$env:USERPROFILE\.claude\projects\*\*.jsonl" `
      -Pattern 'xhci|USBSTOR|Disable-PnpDevice|reg add|bcdedit|bios|flash'

The two patterns worth the most attention are `bcdedit` and anything touching
firmware, since those are the only kinds of change that could plausibly affect
the machine before Windows loads.

## What to check next

Write down two things and the diagnosis narrows sharply: the exact motherboard
model from the silkscreen between the slots, and whether the code stays frozen
on 15 or moves. The jumper location, whether the board has BIOS Flashback, and
the correct single-stick memory slot all depend on the model.
