# dell-hba-sas2008-it-flash

Let’s get this out of the way first: nothing in this repository is new, revolutionary, or even remotely cutting-edge. The newest firmware involved here is well over a decade old—and yet, somehow, still manages to put up a fight.

This repository is a consolidation of tools and steps gathered from various forum posts, guides, and GitHub repositories (all credited below). Think of it less as original research and more as a carefully documented trail of breadcrumbs left behind after a *lot* of trial and error.

I also want to be upfront: I can’t say with absolute certainty that **every** step here is strictly required. What I *can* say is that these are the exact steps that worked for me after multiple false starts, confusing error messages, and more reboots than I care to admit. If you’re here because you’re stuck—this path should get you unstuck.

---

## PREP

You’ll first need to download **Rufus** to create a FreeDOS-bootable USB drive.

Once the USB is created, download the contents of this repository and extract everything directly onto the drive.

The end result is a single USB device that can:

* Boot into **FreeDOS** (for the initial cleanup)
* Boot into a **UEFI shell** (for the firmware flashing)

You’ll switch between both modes on the same USB during this process.

---

## Step 1 – Enter BIOS / Firmware Setup

From Windows (run Command Prompt as Administrator):

```
shutdown /r /fw /t 0
```

From Linux:

```
sudo systemctl reboot --firmware-setup
```

Alternatively, most systems will let you press **F12**, **DEL**, or a similar key during boot to enter the BIOS manually.

---

## Step 2 – BIOS Configuration

In the BIOS, make the following changes:

* **Enable Legacy Boot**
* **Disable Secure Boot**

Leaving Secure Boot enabled can result in the classic and deeply unhelpful:

> `SBR Write Failed – Error code = 8192`

Disabling these two options should allow you to proceed without that particular headache.

---

## Step 3 – FreeDOS (Legacy Boot)

Boot into the **Legacy** (FreeDOS) option for the USB.

If you *don’t* know your HBA’s address, you can run the following (not required in my case, but useful):

```
megacli.exe -AdpAllInfo -aAll -ApplogFile AdaptersInfo1.txt
```

Next, I wiped the SBR and firmware entirely. This *may* be optional, but starting from a clean slate is what finally worked for me and appears to prevent Fusion-mode issues (including Error 8192).

Write an empty SBR:

```
megarec.exe -writesbr 0 sbrempty.bin
```

Then clean the firmware:

```
megarec.exe -cleanflash 0
```

---

## Step 4 – UEFI Shell (Same USB)

Reboot again and return to the boot menu. This time, select the **UEFI** option for the same USB drive.

You should now be in a UEFI shell.

First, re-flash the Dell firmware:

```
sas2flash.efi -o -f DELL_6GBPSAS.FW -b DELL_MPTSAS2.ROM
```

Yes, this seems counterintuitive. Yes, it works.

---

## Downgrade to P7 (The Important Part)

This is the step that most guides either gloss over or mention in passing—usually without including the old flasher you actually need (with the lone exception of a blog post from nearly eight years ago).

To bypass vendor lock issues (thanks, Dell), you must temporarily downgrade the card to **P7 firmware** using an **older flasher (P5)**. Without this step, you’ll almost certainly encounter the classic:

> `D04 – Failed to Validate Mfg Page2`

This downgrade places the card squarely in the pre-vendor-lock era, which is exactly where we need it before moving forward.

Flash P7 using the old flasher:

```
sas2flash5.efi -o -f 2118IT_P7.bin
```

Or, if you intend to boot from drives connected to the card:

```
sas2flash5.efi -o -f 2118IT_P7.bin -b MPTSAS2_P7.ROM
```

You’ll be prompted to confirm flashing **IT over IR**. Say yes.
(The modern flasher refuses to do this. The old one doesn’t argue.)

At this point, the card is safely back in an era before vendor locking became… enthusiastic.

---

## Upgrade to P20 (Final Firmware)

Now that the card is in IT mode, you can safely use the modern flasher to upgrade to P20.

```
sas2flash.efi -o -f 2118IT.BIN
```

Or, with boot ROM:

```
sas2flash.efi -o -f 2118IT.BIN -b mptsas2.rom
```

---

## Restore the SAS Address

Finally, write a SAS address back to the card.

Rules:

* 16-digit hexadecimal value
* Must not match any other HBA in the system

Command:

```
sas2flash.efi -o -sasadd xxxxxxxxxxxxxxxx
```

---

## Final Notes

At this point, you can relax. These cards are surprisingly hard to brick—I rebooted at least a dozen times across different stages over the course of two days, and the card survived every poor decision I made along the way.

If something goes wrong, you can always boot back into FreeDOS, wipe the card again, and retry. Persistence is the real firmware requirement here.

One thing I didn’t test was skipping FreeDOS entirely and wiping the card from UEFI using:

```
sas2flash.efi -o -e 6
```

or

```
sas2flash.efi -o -e 7
```

That *might* work, but this guide documents the path that definitely did.

---

## Credits & References

Huge thanks to the people who originally documented what worked for them—and for keeping the old flasher around long enough for me to find it in the year of our lord 2026:

- https://www.reddit.com/r/homelab/comments/8cjdz7/tutorial_flash_an_h200_to_it_mode_w_uefi_bios/
- https://github.com/lrq3000/lsi_sas_hba_crossflash_guide
- https://www.truenas.com/community/threads/flashing-dell-perc-h310-h200-ibm-m1015-to-lsi-9211-8i-under-uefi-solution-to-%E2%80%9Eerror-cannot-flash-it-firmware-over-ir-firmware%E2%80%9C.80463/

This repository exists because of their groundwork—and because I really didn’t want the next person to lose two days to the same problem.
