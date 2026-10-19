# Add SSD to an established DRAM pool

Sources: MemCache SSD introduction and detailed UBSIO guide in [sources](sources.md). The transport choice remains A3 SDMA, A5 UB or A5 UBoE; SSD is an additional storage backend, not a network protocol.

## Device ownership and memory

Check on the SSD host:

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS
findmnt --source /dev/<TARGET>
```

Also check child partitions, swap, holders and existing consumers before assigning a device. Use only an explicitly allocated exclusive cache device with no mounted filesystem or data to preserve. A partition or loop device is supported by the detailed guide, but is not automatically equivalent in performance to a dedicated NVMe. Expose it to the owner container. Never run partitioning, formatting, `wipefs` or force initialization just to make a launch succeed.

Apply to storage owners, not A3 zero-DRAM clients:

```ini
ock.mmc.local_service.storage.enabled = true
ubsio.disk.path = /dev/<EXCLUSIVE_CACHE_DEVICE>
ubsio.mem.size_in_gb = 10
ubsio.standalone.device_count = <OWNERS_SHARING_DISKS_ON_THIS_HOST>
ubsio.standalone.force_new_disk = false
ubsio.wcache.evict_water_level = 0
```

These are template values requiring release validation. Multiple disk paths use colons. `device_count` is not the disk count or cluster-wide NPU count. For one A3 standalone owner use 1. Every owner must agree on the disk-sharing layout.

UBSIO memory is additional to MemCache DRAM. The detailed guide requires at least 5GB per process for SSD, suggests 10GB for fused deployment and 50GB for standalone, bounded by remaining host memory divided by owner count. The introduction also mentions 80GB for A3: that is not a universal minimum. Use an explicit budget; do not allocate all reported free memory or silently shrink to an unsupported size.

## Resolve documentation conflicts before launch

- The introduction says its release lacks recovery and requires `force_new_disk=true`; the detailed guide says `false` loads compatible disk metadata. The template above preserves existing layout only if supported. Check the installed UBSIO release and disk metadata. If initialization is required, stop and obtain explicit authorization for that identified cache device; do not retry with true automatically. False is not a promise of recovery after a crash.
- The detailed guide spells the scratch setting as both `ubsio.batch_read.standalone.use_scratch_pool` (table) and `ubsio.bdm.batch_read.standalone.use_scratch_pool` (body). It requires scratch buffering on A3 and suggests disabling it elsewhere to avoid a copy. Verify the accepted key in the installed UBSIO parser/defaults and effective startup configuration; do not set both spellings and assume it worked. Keep A3 scratch enabled; if it cannot be verified, do not claim that three-tier deployment is ready.
- An old wheel may not expose newer L2.5, scratch or BDM options. Omit unsupported optimizations rather than treating ignored keys as enabled features.

## Eviction and readback

For a version supporting these Meta settings, a tuning example is:

```ini
ock.mmc.evict_threshold_high = 70
ock.mmc.evict_threshold_low = 60
```

This is not a required replacement for defaults (the introduction lists 90/80). Keep low below high and leave room for a batch allocation plus concurrent requests. The introduction's allocation-size warning is release-specific; verify actual allocator behavior when a large Put fails rather than assuming a disk can compensate for arbitrarily small DRAM.

UBSIO L2.5 is a separate host-memory write cache. With a disk configured, `ubsio.wcache.evict_water_level=0` is the documented way to push writes toward L3 promptly. A high value can keep data in L2.5; an external hit or UBSIO hit then does not prove disk readback. Do not conflate L2.5 usage with MemCache DRAM registration.

Validate store -> DRAM pressure/eviction -> physical SSD write -> repeat request -> SSD readback -> successful request. Increase distinct prefixes to displace HBM before reducing DRAM. Stop on unsupported configuration, disk ownership ambiguity or repeated allocation/read errors; preserve the first failure evidence instead of changing topology or initializing the disk.
