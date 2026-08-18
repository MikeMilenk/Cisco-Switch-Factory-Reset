# Cisco Switch Factory Reset
This guide shows how to reset a **Cisco Catalyst 3650** 48-port switch when you don't know the credentials required to log in. The switch must be physically accessible, as this process requires direct access to the console port and the physical **MODE** button.

**The key concept is:**
Temporarily ignore the existing startup configuration → boot into IOS → erase the startup configuration and VLAN database → restore the default boot behavior → boot normally.

## WARNING: This process deletes the switch configuration and VLAN database. Do not use it on a production switch unless the configuration was backed up or no longer required.

---
## Table of Contents:
**- [1. Establish a Console Connection](#1-establish-a-console-connection)**
**- [2. Enter Bootloader](#2-enter-bootloader)**
**- [3. Initialize the Flash](#3-initialize-the-flash)**
**- [4. Check Bootloader Variables](#4-check-bootloader-variables)**
**- [5. Boot the Switch](#5-boot-the-switch)**
**- [6. Delete the Startup Configuration](#6-delete-the-startup-configuration)**
**- [7. Delete the VLAN Database](#7-delete-the-vlan-database)**
**- [8. Verify the Reset](#8-verify-the-reset)**
**- [9. Enter Bootloader Again](#9-enter-bootloader-again)**
**- [10. Restore the Bootloader Variable](#10-restore-the-bootloader-variable)**
**- [11. Boot Normally](#11-boot-normally)**
**- [12. Quick Summary](#12-quick-summary)**
---


# 1. Establish a Console Connection

Connect the switch to your computer using a Cisco console cable. In my case it was RJ-45 to USB console cable.
![Console cable](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/48814c9af7669ce19c18a4b4d2d474d37e6d60f6/images/Console%20port.jpg)

After connecting the console cable, Windows assigns it a **COM** port. To find it, open: `Device Manager` → `Ports (COM & LPT)`

Find the USB Serial device. In this case: `USB Serial Port (COM4)`

![Windows Device Manager](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/48814c9af7669ce19c18a4b4d2d474d37e6d60f6/images/Device%20mgr%20Port.jpg)


**Open PuTTY and select:**
- Connection type: `Serial`
- Serial line: `COM4`
- Speed: `9600`

![PuTTY config](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/6f408e151e9a48fe12604e9bf8f1c98c9f8885b5/images/PuTTY%20Config.PNG)

Click **Open**.

# 2. Enter Bootloader
The **Bootloader** is a pre-boot program that initializes the switch hardware. It interrupts the normal IOS XE boot process and provides a low-level environment for accessing the flash filesystem and recovering or configuring the switch before the OS loads.

- Power off the switch.
- Press and hold the MODE button.
  ![Mode Button](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/48814c9af7669ce19c18a4b4d2d474d37e6d60f6/images/Mode%20Button.jpg)
- Power on the switch. Keep holding MODE until the `SYS` and `ACTV` LEDs are both amber.
  ![Amber Lights](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/6f408e151e9a48fe12604e9bf8f1c98c9f8885b5/images/Amber%20Lights.JPG)
- Release the **MODE** button.

The normal IOS XE boot process has been interrupted and the switch is now in the Bootloader.

# 3. Initialize the Flash

You should see a prompt similar to:
```bash
  flash_init
  boot

switch:
```
![Bootloader initial prompt](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/48814c9af7669ce19c18a4b4d2d474d37e6d60f6/images/Flash%20init.PNG)

Run:
```bash
flash_init
```

`flash_init` initializes the switch's flash filesystem so that the **Bootloader** can access files stored in flash.
The time required can vary depending on the condition and structure of the flash filesystem and the amount of data that needs to be initialized.

Once the terminal prompt returns, check the filesystem:
```bash
dir flash:
```

You may see files such as:
```bash
cat3k_caa-...
packages.conf
config.text
nvram_config
vlan.dat
pnp-tech-time
```
## Important! Do not delete IOS XE system files such as:
```bash
cat3k_...
packages.conf
dc_profile_dir
```
...and other files required by IOS XE. These are part of the switch operating system.

Followed...
```bash
vlan.dat
config.text
```
...are different: they contain user-specific configuration and are the files relevant to the reset process.

![Flash directories](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/48814c9af7669ce19c18a4b4d2d474d37e6d60f6/images/Dir%20flash%20initial.PNG)

# 4. Check Bootloader Variables

Run:
```bash
set
```

Look for: `SWITCH_IGNORE_STARTUP_CFG=0`
This variable determines whether the switch loads or ignores the existing startup configuration during boot. `0` means the startup configuration is loaded; `1` means it is ignored. When set to `1`, the switch will boot in a clean state without applying the existing configuration.

![SIS_CFG=0](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/48814c9af7669ce19c18a4b4d2d474d37e6d60f6/images/CFG0.PNG)

Change it to `1`:

```bash
SWITCH_IGNORE_STARTUP_CFG=1
```

Verify it:

```bash
set
```

You should now see: `SWITCH_IGNORE_STARTUP_CFG=1`

![SIS_CFG=1](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/48814c9af7669ce19c18a4b4d2d474d37e6d60f6/images/CFG1%20-%201.PNG)

In simple terms:

- `0` = use the existing startup configuration
- `1` = ignore the startup configuration during boot

This does not delete the configuration. It only tells the switch to ignore it for the next boot.

# 5. Boot the Switch

Run:
```bash
boot flash:packages.conf
```

The switch should now boot IOS XE without applying the old startup configuration. However, it may prompt you to enter the initial configuration dialog.
At this stage, we skip the initial configuration:
```Would you like to enter the initial configuration dialog?``` → ```No```
```Would you like to terminate auto install?``` → ```Yes```

![Skipping config dialog](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/9bbe5b2075bf9a5a5fe1afdcf07fedb06acd30ae/images/Startup%20config%20prompt.PNG)

# 6. Delete the Startup Configuration

After IOS loads, enter **privileged mode**:
```bash
enable
```

Check the existing configuration:
```bash
show startup-config
```

The configuration should still exist because ```SWITCH_IGNORE_STARTUP_CFG=1``` only caused the switch to ignore it during boot.

![Original Startup config](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/9bbe5b2075bf9a5a5fe1afdcf07fedb06acd30ae/images/Old%20Startup%20config.PNG)

Now erase it:
```bash
erase startup-config
```
Confirm the deletion when prompted by clicking **Enter**.

# 7. Delete the VLAN Database

The **VLAN database** is stored separately from the startup configuration in the `vlan.dat` file in flash memory.

Delete it with:
```bash
delete flash:vlan.dat
```
Confirm the deletion when prompted.

This removes the **VLAN database**, while `erase startup-config` removes the saved **switch configuration**.

![Config and vlan erasing](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/9bbe5b2075bf9a5a5fe1afdcf07fedb06acd30ae/images/erase%20config%20and%20vlan.jpg)

Commands are highlighted in **red**, while the **yellow** highlights show the confirmation prompts. Select **OK** to confirm the deletion.

I also performed a `reload` at this stage, but it is **not required**. I simply chose to reload the switch before continuing..


# 8. Verify the Reset
In the privileged mode (`enable`), check the startup configuration:
```bash
show startup-config
```
You should see something similar to: `startup-config is not present`

Then check the VLANs:
```bash
show vlan brief
```
The old user-created VLANs should be gone, leaving the default VLAN configuration. The expected result is highlighted in **green**.

![Verifying erased config](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/9bbe5b2075bf9a5a5fe1afdcf07fedb06acd30ae/images/Verif%20after%20reset.PNG)

# 9. Enter Bootloader Again

- Power off the switch.
- Hold the MODE button.
- Power it on. Wait until `SYS` and `ACTV` are amber.
- Release the **MODE** button.

At the Bootloader prompt, initialize flash again:
```bash
flash_init
```

Then:
```bash
dir flash:
```
`vlan.dat` and the old configuration files (such as `config.text` or `nvram_config`) should no longer be present.

The IOS XE system files such as:
```bash
cat3k_...
packages.conf
```
...should remain.

![Flash directories after resetting the configs](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/9bbe5b2075bf9a5a5fe1afdcf07fedb06acd30ae/images/dir%20flash.PNG)

# 10. Restore the Bootloader Variable

Check the current variables:
```bash
set
```

You should still see:
  ```SWITCH_IGNORE_STARTUP_CFG=1```
![Still existing SIS_CFG=1](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/9bbe5b2075bf9a5a5fe1afdcf07fedb06acd30ae/images/CFG1%20-%202.jpg)

Now return it to the normal value:

```bash
SWITCH_IGNORE_STARTUP_CFG=0
```

Verify:
```bash
set
```

It should now show:
  ```SWITCH_IGNORE_STARTUP_CFG=0```

# 11. Boot Normally

If the boot variable is: ```BOOT=flash:packages.conf``` you can simply run:
```bash
boot
```
or explicitly:
```bash
boot flash:packages.conf
```

![Final SIS_CFG=0](https://github.com/MikeMilenk/Cisco-Switch-Factory-Reset/blob/9bbe5b2075bf9a5a5fe1afdcf07fedb06acd30ae/images/Final%20CFF0.jpg)

Skip or accept the Initial Configuration Dialog

After boot, the switch may ask: ```Would you like to enter the initial configuration dialog? [yes/no]:```

# 12. Quick Summary

The complete process is:

**Bootloader**
- `flash_init`
- `dir flash:`
- `set`
- `SWITCH_IGNORE_STARTUP_CFG=1`
- `boot`

**IOS**
- `enable`
- `show startup-config`
- `erase startup-config`
- `delete flash:vlan.dat`

**Verify**
- `enable`
- `show startup-config`
- `show vlan brief`

**Bootloader again**
- `flash_init`
- `dir flash:`
- `set`
- `SWITCH_IGNORE_STARTUP_CFG=0`
- `boot`
