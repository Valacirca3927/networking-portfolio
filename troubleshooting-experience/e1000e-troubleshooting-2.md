# Problem Description
Follow-up from [previous case](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting.md), Proxmox nodes still becoming isolated from network due to host NIC driver hanging

# Action Plan
Change host node configs to manage remote mountpoint inodes themselves.  
`sudo pvesm set unraid-isos --options noserverino`  
`sudo pvesm set unraid-vm-backup --options noserverino`

Reboot.

# Previous Casework
[Prior context](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting.md)

# Valerie's Work
Docker VM hangs with same symptoms as previous case. Host machine e1000e driver crashes, node becomes isolated, services unreachable.

No obvious precipitating event. Reading logs around both incidents shows only one similarity, a several-hour period of remote mountpoints being unreachable.

Checking mountpoints shows stale file handle. Research shows this is known issue with SMB shares and Linux. Server restarts, uses different inode, clients try to reach an inode that no longer exists. 

No other events within 48h of driver failure.

e1000e driver is buggy, nothing else out of the ordinary, repeated tx failures are likely candidate for driver crashing.

Disabling hardware offloading on transmit is next community approved way to handle this, but still instructed to avoid that. Finding alternative solutions.

Clients can use `noserverino` option to keep track of inode values themselves, preventing spoilage.

Stale file handles are likely root cause of both crashes, preventing them should reduce chances for tx failures to pile up and crash the driver.

Implementing `noserverino` option on host node mountpoints and continuing to monitor.
