# lvm-thin-sendrcv
Send and receive incremental / thin LVM snapshots.  Replication / synchronisation of an LVM volume to a remote server by transmitting only the difference between snapshots.

# Status
Working in an initial form.  Needs a little setup, documented at [LVMSnapSend.md](LVMSnapSend.md).  Remember, you must **never write to the target volume**.

Monitor your thin LVs using [LVMSpaceCheck](LVMSpaceCheck.md)
Also useful is [DDRandom.md](DDRandom.md)

# Purpose 
LVM volumes (LVs) are useful, however synchronising them between machines is generally slow, especially when repeated.  The goals of this project are:
- Synchronise LVs between machines similar to a zfs send / receive
- No need to read the entire LV to run a sync (low IO)
- Does not have a large performance penalty in either reads, writes, or CPU.
- Able to verify and mount the snapshots 
- No need to unmount the volume / shut down the virtual machine
- Filesystem and encryption agnostic.  I should be able to sync ext4, xfs, etc.

The use case I am aiming for is to have VM images synchronised across to another server.

# Prerequisites / Platform
This is only intended to run on thinly provisioned LVM.

I will be testing on Ubuntu, but welcome feedback for other platforms / OSes.

# Goals
~~This is literally the start of the project.  So far there is nothing.  ~~Goals for the project are:
- ~~Write useful libraries to snapshot the LV, snapshot the thin metadata, and extract the changed blocks and block sizes~~
- ~~Basic proof of concept.  Synchronise two manually created snapshots between 2 servers.  Probably using dd over ssh or something else lame.~~
- ~~Extract things like block size and thin volume using lvs and co.~~
- ~~Create and delete the snapshots as well as synchronise~~
- Server mode over ssh
- Run as a daemon so that your LVs are continuously synchronised.  This requires robust error handling
- Alerts / monitoring / emails
- Ensure continuous consistency at the destination e.g. when there is a failure partway through the sync.  Probably by snapshotting at the destination.
- Allowing for synchronising to a file.  This would be VERY useful.
- Verify the snapshot at both ends (probably daily, as otherwise we are reading the LV every time)
- Automate the original send (something like blocksync) and possibly LV or file creation.
- Alerting on thin pool size and metadata usage
- World domination!

There is absolutely no intent to support synchronising without a snapshot.

# Performance
Running on a single AWS small instance and via SSH, I am getting changes detected, read, transferred, and written at roughly 10MB/s.  This means that for 50MB of writes, it takes ~5 seconds to do a sync.  This is irrespective of the size of the underlying device.  For devices in the 10s or 100s of GB this is a boon.

Running rsyncs of large file collections in the 10s and 100s of MB total e.g. `/usr/bin` or `/usr/lib` the size of changed blocks is very close to the size of the files.  In other words, 10MB of files leads to 11 - 12 MB of block changes.

This performance makes it seem reasonable that we could potentially be synchronising the device every minute (or more frequently) once a service has been written.

# Alternatives
- Blocksync https://github.com/theraser/blocksync is very good but reads the entire LV.  For large LVs this is **slow** and consumes IO on the source and target
- lvsync https://github.com/mpalmer/lvmsync seems to have a lot of features **but** requires the LV to be closed / VM to be shut down.  It also does an initial full sync just like blocksync.
- drdb https://www.linbit.com/ is costly if you want the proxy (and if running over a WAN, you need a proxy) and there is no easy way to verify the replicas as the other end
- sparsebak https://github.com/tasket/sparsebak seems to be aimed at backups rather than replication

# Sponsors
This project is sponsored by:
- [OneIT Custom Software](https://www.oneit.com.au)
- [Secure Hosting Australia](https://www.secure-hosting.com.au)

# Technical approach
Thin LVM is not the most well documented API.  The high level approach for now is:

I will be maintaining at least a basic LVM document to outline how thing LVM snapshotting works as it seems like a useful service.
```
# Get the chunk size
lvs -o lv_name,chunksize volg/volg-thinpool

# Get the device IDs
SNAP1_ID=$(lvs --noheadings -o thin_id volg/thin_volume_snap4)
SNAP2_ID=$(lvs --noheadings -o thin_id volg/thin_volume_snap5)

# Reserve the metadata
dmsetup message /dev/mapper/volg-volg--thinpool-tpool 0 reserve_metadata_snap
dmsetup status /dev/mapper/volg-volg--thinpool-tpool  # Get the snapshot ID piping through: cut -f 7 -d " "

# Determine the difference between the snapshots
thin_delta  -m --snap1 $SNAP1_ID --snap2 $SNAP2_ID /dev/mapper/volg-volg--thinpool_tmeta

# Release the metadata snapshot.  Try to keep this window short.
dmsetup message /dev/mapper/volg-volg--thinpool-tpool 0 release_metadata_snap

# Take those blocks and push them to the target device / file.
```

