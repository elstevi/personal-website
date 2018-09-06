<!--
.. title: FreeBSD replace a root disk from bsdinstall connected to a RAID controller
.. slug: freebsd-replace-a-root-disk-from-bsdinstall-connected-to-a-raid-controller
.. date: 2018-03-27 22:51:26 UTC-07:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

I have a failed drive in my pool!
```
# mfiutil show drives
mfi0 Physical Drives:
 0 (  137G) FAILED <FUJITSU MBE2147RC D905 serial=D304PB40ASPD> SAS E1:S0
 1 (  932G) ONLINE <MM1000GBKAL HPGB serial=9XG259SP> SATA E1:S1
```

<!-- TEASER_END -->

I just replaced the old bad drive with a shiny new (larger) one:

```
# mfiutil show drives
mfi0 Physical Drives:
 0 (  932G) UNCONFIGURED GOOD <MM1000GBKAL HPGC serial=9XG271GS> SATA E1:S0
 1 (  932G) ONLINE            <MM1000GBKAL HPGB serial=9XG259SP> SATA E1:S1
```

And finally create a new raid0 to pass this disk through:
```
# mfiutil create raid0 1
mfiutil: Command failed: Status: 0x54
mfiutil: Failed to add volume: Input/output error
```
Uh oh. After installing `sysutils/megacli` from ports, eventually I figured out that this was an issue with data in the controller cache:

```
# MegaCli -CfgLdAdd -r0 [32:1] -a0


Adapter 0: Configure Adapter Failed

FW error description:
  The current operation is not allowed because the controller has data in cache for offline or missing virtual drives.

Exit Code: 0x54
```

32 is my adapter ID. I found that in the LSI Controller list. Now we can discard that cache data:

```
# MegaCli -DiscardPreservedCache -L0 -a0
Adapter #0
Virtual Drive(Target ID 00): Preserved Cache Data Cleared.
Exit Code: 0x00
```

Now:
```
# mfiutil create raid0 1
```
Returns normally.


Now you can partition the GPT disk (I am ommitting the swap partition):
```
# gpart create -s gpt /dev/mfid0
# gpart add -t freebsd-boot -s 512K mfid0
# gpart add -t freebsd-zfs -A 1M mfid0
# gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 mfid0
```

And replace the disk:

```
# zpool replace zroot 5429029324314476965 /dev/mfid0p2
# zpool status zroot
  pool: zroot
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Thu Nov 23 23:43:26 2017
        999M scanned out of 55.5G at 22.2M/s, 0h41m to go
        999M resilvered, 1.76% done
config:

        NAME                       STATE     READ WRITE CKSUM
        zroot                      DEGRADED     0     0     0
          mirror-0                 DEGRADED     0     0     0
            replacing-0            UNAVAIL      0     0     0
              3351751284279945146  UNAVAIL      0   296     0  was /dev/mfid0p3
              mfid0p2              ONLINE       0     0     0  (resilvering)
            mfid1p2                ONLINE       0     0     0
```

Eventually, when the resilver finishes, you will be left with this:
```
# zpool status zroot
  pool: zroot
 state: ONLINE
  scan: resilvered 55.5G in 0h28m with 0 errors on Fri Nov 24 00:11:52 2017
config:

        NAME         STATE     READ WRITE CKSUM
        zroot        ONLINE       0     0     0
          mirror-0   ONLINE       0     0     0
            mfid0p2  ONLINE       0     0     0
            mfid1p2  ONLINE       0     0     0

errors: No known data errors
``` 
