# Problem Description
Proxmox node isolated from network due to host NIC driver hanging under load. 
(?) Potentially incorrect default kernel driver settings.

# Action Plan
Set better kernel settings manually.  
`echo "options e1000e InterruptThrottleRate=3" > /etc/modprobe.d/e1000e.conf`

Reload driver, then network stack. Connection loss of about a minute.  
`(modprobe -r e1000e && modprobe e1000e; sleep 30; systemctl restart networking) &`

**Verify.** 
InterruptThrottlingRate should be set to dynamic conservative.  
`dmesg | grep -i "e1000e" | grep -i "interrupt"`

# Previous Engineer
* Replaced Motherboard (with NIC), problem unresolved

# Valerie's Work
**Note:** This issue has been recurring.

**Docker VM 2 unreachable, network services degraded.**

Proxmox reports Docker VM 2 hung during backup session, its host is unreachable. Nebula Sync reports inability to sync DNS databases.

Console to host machine shows error flooding the terminal:  
`e1000e 0000:00:19.0 eno1: Detected Hardware Unit Hang:`

e1000e driver has multiple unresolved bugs spanning multiple Proxmox kernel versions. No permanent fix.

Community-suggested workaround disables hardware offloading. Tech Lead has declared undesirable. Finding another workaround.

Alternative workaround is setting `InterruptThrottleRate` parameter to dynamically scale down the errors sent to the CPU. Workaround will need monitoring, as issue is intermittent.

Switching heads didn't work to get around the error flood.  
Stop errors from displaying on terminal:  
`sudo dmesg -n 1`  
(This just disables throwing every message at the terminal, it will still be stored in the logs.)

Restart interface, clearing errors and re-enabling SSH:  
`ip link set eno1 down`  
`ip link set eno1 up`

Proxmox can see host machine now, VM still says config locked.

Workaround:  
`echo "options e1000e InterruptThrottleRate=3" > /etc/modprobe.d/e1000e.conf`

Reload driver:  
`modprobe -r e1000e && modprobe e1000e`  
SSH drops, doesn't reestablish.

Reload networking stack:  
`systemctl restart networking`  
SSH connection reestablishes after about a minute.

Verify change to dynamic conservative:  
`dmesg | grep -i "e1000e" | grep -i "interrupt"`

Manually unlock the VM:  
`qm unlock 103`  
Proxmox says VM is running normally. Nebula Sync container reports DNS databases are now synched.

Set up monitor:  
`watch -n 2 'dmesg -T | grep -i "hardware unit hang" | tail -3'`

Trigger backup job:  
`vzdump 103 --storage unraid-vm-backup --mode snapshot --compress zstd --bwlimit 716800`

Backup completed successfully, no errors. Proxmox sees backup file in GUI. All symptoms currently resolved.

Workaround needs monitoring. If issue recurs, next steps may include disabling hardware offloading and accepting the CPU hit to keep services up.

Issue does recur and troubleshooting continues [here.](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting-2.md)



