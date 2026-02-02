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

## Execution Flow

The capability is restored **immediately after** the UID change, before any
filesystem syscall. Here's the sequence for a file operation:

```
1. Guest requests file open (e.g., cat /mnt/nfs/file.txt)
2. virtiofsd calls UnixCredentials::set()
3.   → setresuid(-1, uid, -1)           // euid: 0 → 1000, loses capabilities
4.   → add_cap_to_eff("DAC_READ_SEARCH") // restore capability immediately
5. virtiofsd calls open_by_handle_at()   // succeeds (has capability)
6. File operation completes
7. UnixCredentialsGuard is dropped
8.   → setresuid(-1, 0, -1)             // euid: 1000 → 0, caps auto-restored
```

The `UnixCredentialsGuard` is a Rust RAII guard that automatically resets the
UID back to root when it goes out of scope. This ensures credentials are always
properly cleaned up, even if an error occurs.

## Multiple Operations

The patch works for **every** file operation, not just once. Each operation
creates a new credential guard, and the capability is restored each time:

```
First read:
  set() → setresuid(0→1000) → add CAP → open → drop guard → back to root

Second read:
  set() → setresuid(0→1000) → add CAP → open → drop guard → back to root

Nth read:
  ... same flow, capability restored every time
```

The patch is in `UnixCredentials::set()`, which is called for every file
operation that requires credential switching:

```rust
pub fn set(self) -> io::Result<Option<UnixCredentialsGuard>> {
    // ...
    if change_uid {
        oslib::seteffuid(self.uid)?;                      // UID change
    }

    if change_uid {
        crate::util::add_cap_to_eff("DAC_READ_SEARCH");   // restore cap
    }

    Ok(Some(UnixCredentialsGuard { ... }))                // return guard
}
```

Since `set()` is called for each operation, the capability is always available
when needed. The guard handles cleanup only (resetting UID back to root).

## UID Independence

The fix works with **any container UID** (999, 1000, 65534, etc.). The patch
restores the capability after any UID change - it doesn't matter what the
target UID is.

File ownership on NFS depends on the export configuration:

| Container UID | all_squash (anonuid=999) | root_squash |
|---------------|--------------------------|-------------|
| 999 | Files owned by 999 | Files owned by 999 |
| 1000 | Files owned by 999 | Files owned by 1000 |
| 65534 | Files owned by 999 | Files owned by 65534 |

## NFS Export Configurations

The fix works with all common NFS export options:

| Export Option | Behavior | Supported |
|---------------|----------|-----------|
| `all_squash` | Maps all UIDs to anonymous user | Yes |
| `root_squash` | Maps only root to anonymous, others pass through | Yes |
| `no_squash` | All UIDs pass through as-is | Yes |

## Current Limitation: Group-Based Write Access

The current fix (`CAP_DAC_READ_SEARCH`) only supports **owner-based access**. Group-based
write access does NOT work.

### What Works

| Access Type | READ | WRITE |
|-------------|:----:|:-----:|
| Owner (uid matches) | ✅ | ✅ |
| Group (gid matches, uid differs) | ✅ | ❌ |
| Other | ❌ | ❌ |

### Test Results

With NFS export `all_squash,anonuid=1000,anongid=1000` and directory owned by `1000:1000`:

| User | uid:gid | WRITE | READ | Error |
|------|---------|:-----:|:----:|-------|
| appnfs | 1000:1000 | ✅ | ✅ | - |
| appnfs2 | 1001:1000 | ❌ | ✅ | EPERM |
| appother | 2000:1234 | ❌ | ❌ | Permission denied |

### Why Group Write Fails

Two capabilities are relevant:

| Capability | Purpose | Current Status |
|------------|---------|----------------|
| `CAP_DAC_READ_SEARCH` | Bypass read permission checks | ✅ Added unconditionally |
| `CAP_DAC_OVERRIDE` | Bypass write permission checks | ⚠️ Conditional on `keep_capability` flag |

The code already has `CAP_DAC_OVERRIDE` but it's gated:

```rust
if change_uid {
    add_cap_to_eff("DAC_READ_SEARCH");  // always added
}

if change_uid && self.keep_capability {  // only if keep_capability=true
    add_cap_to_eff("DAC_OVERRIDE");
}
```

### Potential Fix

To enable group-based write access, either:
1. Enable `keep_capability` flag in virtiofsd configuration
2. Add `CAP_DAC_OVERRIDE` unconditionally (like `CAP_DAC_READ_SEARCH`)

**Security note**: `CAP_DAC_OVERRIDE` bypasses ALL write permission checks, which may
have security implications. The `keep_capability` flag exists for this reason.

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
