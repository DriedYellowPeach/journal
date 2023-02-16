Here are problem need to further considered:
- when restore container, if find image not exist, will delete the rw rootfs, this is bad when have current network failure.
- the daemon.json root path need to be reset, the RO directory and 
- delete image by accident, what's the consaquence.
