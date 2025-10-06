# Final Problem Description
Hypervisor was crashing because the host machine e1000e driver was pegging the CPU with errors under high transmit load. Potentially incorrect default kernel driver settings.

# Action Plan
Set correct kernel settings manually.  
`echo "options e1000e InterruptThrottleRate=3" > /etc/modprobe.d/e1000e.conf`

Reload driver, then network stack. Connection loss of about a minute.  
`(modprobe -r e1000e && modprobe e1000e; sleep 30; systemctl restart networking) &`

**Verify.** InterruptThrottlingRate should be set to dynamic conservative.  
`dmesg | grep -i "e1000e" | grep -i "interrupt"`

# Incidents
## Oct 5, 2025
#### Docker VM 2 and Containerized Services down, Host Machine Unreachable
**Note:** This issue has been reoccuring. Case was passed to me by previous engineer after hardware replacement failed to fix the issue.

Initial Issue: Docker VM 2 and associated services show as down in Portainer, unreachable directly. Proxmox GUI reports the VM hung during a backup session and is stuck in that state. Proxmox GUI and ping report the host machine is unreachable. Nebula Sync container reports unhealthy state due to inability to sync the DNS databases to each other.

Console access to host machine shows an error flooding the terminal:  
`e1000e 0000:00:19.0 eno1: Detected Hardware Unit Hang:`

A search reveals that this is a long-standing, unresolved driver issue with Intel chipsets spanning multiple Proxmox kernel versions (5.x through 6.8.x as of October 2025). No permanent fix exists, there are multiple bug threads for this driver, multiple fail-states, and the issue crops up again in regressions.

The frequency of the errors suggests that the particular failure state we're hitting *may* be resolved by setting the InterruptThrottleRate=3 parameter, which should dynamically scale down errors sent to the CPU when too many are thrown in a short time, as documented [here.](https://www.kernel.org/doc/Documentation/networking/e1000e.txt)

Effectiveness will need to be monitored over time, as the issue is intermittent. If it persists, an alternative community-suggested workaround would involve disabling hardware offloading to further reduce the CPU load, but this would slow the network connection, so it is a last resort.

#### First step: Stop console from being flooded
Switching heads didn't work to get around the message flood.  
`sudo dmesg -n 1` worked, enabling further troubleshooting.  
(This just disables throwing every message at the terminal, it will still be stored in the logs.)

#### Second step: Restart interface to stop errors
Ran console commands to restart the interface, which should clear the errors and re-enable SSH:  
`ip link set eno1 down`  
`ip link set eno1 up`

SSH works, and Proxmox can see the host machine now, but still shows the VM as config locked because of a backup.

#### Third step: Implement proposed workaround
Confirmed no `e1000e.conf` file existed.  
Added it with the missing parameter `echo "options e1000e InterruptThrottleRate=3" > /etc/modprobe.d/e1000e.conf`

Reloaded just the driver with `modprobe -r e1000e && modprobe e1000e`  
SSH dropped, would not reestablish. Switched back to console.

Reloaded the full networking stack with `systemctl restart networking`  
SSH connection reestablished after about a minute.

#### Fourth step: Resolve VM config lock
Read VM logs in the Proxmox GUI, noted that the backup proceeded to 100%, but failed to exit properly, so it never unlocked or registered the backup as valid.

Manually unlocked the VM via `qm unlock 103`  
Proxmox now sees the VM running normally. Nebula Sync container has returned to a healthy state, reporting DNS databases are synched.

#### Fifth step: Perform the backup job that originally crashed the hypervisor as a test
Set up a `watch -n 2 'dmesg -T | grep -i "hardware unit hang" | tail -3'` command on console to keep an eye on errors in case SSH was lost again.

Triggered a backup with `vzdump 103 --storage unraid-vm-backup --mode snapshot --compress zstd --bwlimit 716800` to mimic the existing VM settings. 

Started documenting while waiting.

Backup completed successfully with no perceptible difference in speed. Proxmox sees the new backup file in the GUI. All symptoms currently resolved.

#### Watchful waiting:
Workaround needs to be validated over time. If the issue recurs, next steps can include disabling offloading features and accepting the network performance hit for the sake of stability.
