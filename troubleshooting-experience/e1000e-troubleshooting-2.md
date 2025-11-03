# Problem Description
Follow-up from [previous case](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting.md), Proxmox nodes still becoming isolated from network due to host NIC driver hanging

# Action Plan
Change host node configs to manage remote mountpoint inodes themselves.  

```
sudo pvesm set unraid-isos --options noserverino  
sudo pvesm set unraid-vm-backup --options noserverino
sudo reboot
```

Reboot.

# Previous Casework
[Prior context](https://github.com/Valacirca3927/networking-portfolio/blob/main/troubleshooting-experience/e1000e-troubleshooting.md)

# Valerie's Work
Docker VM hangs with same symptoms as previous case. Host machine e1000e driver crashes, node isolated, services unreachable.

No known cause. Logs show remote mountpoints unreachable during both incidents.

Accessing mountpoints shows stale file handle. This is a known issue with SMB shares and Linux. Server restarts, uses different inode, clients try to reach inode that no longer exists. 

No other events within 48h of driver failure.

e1000e driver is buggy, nothing else out of ordinary, repeated tx failures likely candidate for driver crashing.

Disabling hardware offloading on transmit is next community approved workaround. Tech Lead has declared undesirable. Finding another workaround.

Clients can use `noserverino` option to keep track of inode values themselves, preventing spoilage.

Stale file handles likely root cause of both crashes, preventing should reduce chances for tx failures to crash the driver.

Implementing `noserverino` option on host node mountpoints, continuing to monitor.
