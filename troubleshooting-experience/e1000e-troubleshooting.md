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
**Note:** This issue has been reoccuring.

**Docker VM 2 unreachable, network services degraded.**

Proxmox reports Docker VM 2 hung during backup session, its host is unreachable. Nebula Sync reports inability to sync DNS databases.

Console to host machine shows an error flooding the terminal:  
`e1000e 0000:00:19.0 eno1: Detected Hardware Unit Hang:`

Online reports show the e1000e driver has multiple unresolved bugs spanning multiple Proxmox kernel versions. No permanent fix.

A community-suggested workaround disables hardware offloading, but works the CPU harder and potentially bottlenecks bandwidth. Exploring alternatives.

An alternative workaround is setting the InterruptThrottleRate parameter to dynamically scale down the errors sent to the CPU. Effectiveness will need to be monitored over time, as the issue is intermittent.

Switching heads didn't work to get around the error flood.  
Stop errors from displaying on terminal:  
`sudo dmesg -n 1`  
(This just disables throwing every message at the terminal, it will still be stored in the logs.)

Restart the interface, clearing the errors and re-enabling SSH:  
`ip link set eno1 down`  
`ip link set eno1 up`

Proxmox can see the host machine now, VM still says config locked.

Workaround:  
`echo "options e1000e InterruptThrottleRate=3" > /etc/modprobe.d/e1000e.conf`

Reload driver:  
`modprobe -r e1000e && modprobe e1000e`  
SSH drops, doesn't reestablish.

Reload networking stack:  
`systemctl restart networking`  
SSH connection reestablishes after about a minute.

Verify setting change to dynamic conservative:  
`dmesg | grep -i "e1000e" | grep -i "interrupt"`

Manually unlock the VM:  
`qm unlock 103`  
Proxmox says VM is running normally. Nebula Sync container reports DNS databases are now synched.

Set up monitor:  
`watch -n 2 'dmesg -T | grep -i "hardware unit hang" | tail -3'`

Trigger backup job:  
`vzdump 103 --storage unraid-vm-backup --mode snapshot --compress zstd --bwlimit 716800`

Backup completed successfully, no errors thrown. Proxmox sees backup file in the GUI. All symptoms currently resolved.

Workaround needs validation over time. If issue recurs, next steps can include disabling hardware offloading and accepting the CPU hit to keep services up.

Issue does recur and troubleshooting continues [here.](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting-2.md)


