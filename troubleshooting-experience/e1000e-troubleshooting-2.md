# Problem Description
Follow-up from [previous case](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting.md), Proxmox nodes still becoming isolated from network due to host NIC driver hanging

# Action Plan
Change host node configs to keep track of remote mountpoint inodes themselves instead of relying on the server.  
`pvesm set <storage> --options noserverino`

Reboot for changes to take effect.

# Previous Casework
[Prior context](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting.md)

# Valerie's Work
Docker VM hangs with the same symptoms as previous case. Host machine e1000e driver crashes, node becomes isolated, services unreachable.

No obvious precipitating event. Digging into the logs for both incidents shows only one similarity, an extended period of one of the remote mountpoints being unreachable.

Investigating the mountpoints shows a stale file handle. Research says this is a common problem with accessing SMB shares via Linux. The server restarts, uses a different inode, clients try to reach an inode that no longer exists. 

There are no other visible errors or events within 48h of driver failure, besides the repeated failures to access the remote mountpoint.

The e1000e driver is notably buggy. With nothing else out of the ordinary, the repeated tx failures are a likely candidate for the driver crashing.

Disabling hardware offloading on transmit is the next community approved way to handle this, but we still want to avoid that. Investigating alternative solutions.

Clients can be told to use the `noserverino` option to keep track of inode values themselves rather than using the server provided values, preventing spoilage.

Stale file handles are the likely root cause of both crashes, preventing them should greatly reduce chances for tx failures to pile up and crash the driver.

Implementing the `noserverino` option on host node SMB mountpoints and continuing to monitor.
