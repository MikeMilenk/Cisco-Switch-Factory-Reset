# Cisco Switch Factory Reset
This guide shows how to reset a **Cisco Catalyst 3650** 48-port switch when you don't know the credentials required to log in. The switch must be physically accessible, as this process requires direct access to the console port and the physical **MODE** button.

**The key concept is:**
Temporarily ignore the existing startup configuration → boot into IOS → erase the startup configuration and VLAN database → restore the default boot behavior → boot normally.

## WARNING: This process deletes the switch configuration and VLAN database. Do not use it on a production switch unless the configuration was backed up or no longer required.


# 1. Establish a Console Connection

Connect the switch to your computer using a Cisco console cable. In my case it was RJ-45 to USB console cable.
After connecting the console cable, Windows assigns it a **COM** port. To find it, open: `Device Manager` → `Ports (COM & LPT)`

Find the USB Serial device. In this case: ```USB Serial Port (COM4)```

Open PuTTY and select:
- Connection type: `Serial`
- Serial line: `COM4`
- Speed: `9600`
Click **Open**.

# 2. Enter Bootloader
The Bootloader is a pre-boot program that initializes the switch hardware. It interrupts the normal IOS XE boot process and provides a low-level environment for accessing the flash filesystem and recovering or configuring the switch before the OS loads.

- Power off the switch.
- Press and hold the MODE button.
- Power on the switch. Keep holding MODE until the `SYS` and `ACTV` LEDs are both amber.
- Release the **MODE** button.

The normal IOS XE boot process has been interrupted and the switch is now in the Bootloader.

# 3. Initialize the Flash

You should see a prompt similar to:

```bash
  flash_init
  boot

switch:
```

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
cat3k_...
packages.conf
config.text
nvram_config
vlan.dat
pnp-tech-time
```
## Important: Do not delete IOS XE system files such as:
```bash
cat3k_...
packages.conf
dc_profile_dir
```
...and other files required by IOS XE. These are part of the switch operating system.

`vlan.dat` and `config.text` are different: they contain user-specific configuration and are the files relevant to the reset process.

# 4. Check Bootloader Variables

Run:
```bash
set
```

Look for:
```bash
SWITCH_IGNORE_STARTUP_CFG=0
```
This variable determines whether the switch loads or ignores the existing startup configuration during boot. `0` means the startup configuration is loaded; `1` means it is ignored. When set to `1`, the switch will boot in a clean state without applying the existing configuration.

Change it to `1`:

```bash
SWITCH_IGNORE_STARTUP_CFG=1
```

Verify it:

```bash
set
```

You should now see: `SWITCH_IGNORE_STARTUP_CFG=1`

In simple terms:

- `0` = use the existing startup configuration
- `1` = ignore the startup configuration during boot

This does not delete the configuration. It only tells the switch to ignore it for the next boot.

# 5. Boot the Switch

Run:
```bash
boot flash:packages.conf
```

If the boot variable points to `flash:packages.conf`, you can simply run:
```bash
boot
```
The switch should now boot IOS XE without applying the old startup configuration.


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

## 8. Verify the Reset
In the privileged mode (`enable`), check the startup configuration:
```bash
show startup-config
```
You should see something similar to:
```bash
startup-config is not present
```
Then check the VLANs:
```bash
show vlan brief
```
The old user-created VLANs should be gone, leaving the default VLAN configuration.

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
should remain.

# 10. Restore the Bootloader Variable

Check the current variables:
```bash
set
```

You should still see:
```bash
SWITCH_IGNORE_STARTUP_CFG=1
```

Now return it to the normal value:

```bash
SWITCH_IGNORE_STARTUP_CFG=0
```

Verify:
```bash
set
```

It should now show: ```SWITCH_IGNORE_STARTUP_CFG=0```

# 11. Boot Normally

If the boot variable is: ```BOOT=flash:packages.conf``` you can simply run:
```bash
boot
```
or explicitly:
```bash
boot flash:packages.conf
```

# 12. Skip the Initial Configuration Dialog

After boot, the switch may ask: `Would you like to enter the initial configuration dialog? [yes/no]:`

Enter:
```bash
no
```

# 13. Quick Summary

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
