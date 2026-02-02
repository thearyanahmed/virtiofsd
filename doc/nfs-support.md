# NFS Support

This document describes the changes required for virtiofsd to work correctly
with NFS-backed filesystems.

## Problem

When virtiofsd shares an NFS mount with a guest VM, file read operations fail
with `EPERM (Operation not permitted)` even when:
- Write/create operations succeed
- File ownership is correct
- File permissions allow reading

This affects both `all_squash` and `root_squash` NFS export configurations.

## Root Cause

NFS files are accessed via file handles using the `open_by_handle_at()` syscall.
This syscall requires `CAP_DAC_READ_SEARCH` capability.

When virtiofsd changes the effective UID from root to the guest user via
`setresuid()`, it loses effective capabilities including `CAP_DAC_READ_SEARCH`.
The subsequent `open_by_handle_at()` call then fails with EPERM.

From `man 2 open_by_handle_at`:

> CAP_DAC_READ_SEARCH
>     Bypass file read permission checks and directory read and execute
>     permission checks.

## Solution

Add `CAP_DAC_READ_SEARCH` back to the effective capability set after the UID
change in `src/passthrough/credentials.rs`:

```rust
if change_uid {
    oslib::seteffuid(self.uid)?;
}

if change_uid {
    // add DAC_READ_SEARCH capability to allow open_by_handle_at syscall
    // this capability is required when using file handles (which NFS uses)
    if let Err(e) = crate::util::add_cap_to_eff("DAC_READ_SEARCH") {
        warn!("failed to add 'DAC_READ_SEARCH' to the effective set of capabilities: {e}");
    }
}
```

This is safe because:
1. We only modify the effective capability set, not the permitted set
2. The capability is already in the permitted set (virtiofsd runs as root)
3. When switching back to root, the permitted set is copied to effective

## NFS Export Configurations

The fix works with all common NFS export options:

| Export Option | Behavior | Supported |
|---------------|----------|-----------|
| `all_squash` | Maps all UIDs to anonymous user | Yes |
| `root_squash` | Maps only root to anonymous, others pass through | Yes |
| `no_squash` | All UIDs pass through as-is | Yes |

## Testing

Verify NFS operations work correctly:

```bash
# from inside the guest/container
touch /mnt/nfs/test.txt          # CREATE
echo "hello" > /mnt/nfs/test.txt # WRITE
cat /mnt/nfs/test.txt            # READ
echo "world" >> /mnt/nfs/test.txt # APPEND
mkdir /mnt/nfs/testdir           # MKDIR
rm /mnt/nfs/test.txt             # DELETE
```

## Related

- Linux capabilities: `man 7 capabilities`
- open_by_handle_at: `man 2 open_by_handle_at`
