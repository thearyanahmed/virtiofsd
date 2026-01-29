# virtiofsd NFS root_squash Credentials Fix

## Problem

When using virtiofsd with NFS storage that has `root_squash` enabled, file operations fail or create files with wrong ownership.

### Root Cause

virtiofsd runs as root on the host. NFS `root_squash` maps root (uid=0) to nobody (uid=65534) for security. The issue is that virtiofsd's passthrough filesystem implementation doesn't set the correct credentials before certain operations:

- `open()` - used `_ctx: Context` (unused)
- `write()` - used `_ctx: Context` (unused)
- `unlink()` - used `_ctx: Context` (unused)

This means these operations execute with fsuid=0 (root), which NFS maps to nobody:nogroup.

### Symptoms

```
# file created by container user (uid=999)
-rw-r--r-- 1 nobody nogroup 13 Jan 29 12:00 myfile.txt

# expected
-rw-r--r-- 1 nfsb nfsb 13 Jan 29 12:00 myfile.txt
```

Operations like write, append, and delete fail with "Permission denied" because the container user (999) cannot modify files owned by nobody (65534).

## Solution

Patch virtiofsd to call `unix_credentials_guard()` before filesystem operations, which sets the correct fsuid/fsgid via `setfsuid()`/`setfsgid()` system calls.

### Patched Functions

1. **`fn open()`** - Set credentials before opening files
2. **`fn write()`** - Set credentials before writing
3. **`fn unlink()`** - Set credentials before deleting

### Patch Location

`src/passthrough/mod.rs`

### Example Change (open function)

```rust
// BEFORE
fn open(
    &self,
    _ctx: Context,  // UNUSED - no credentials set
    inode: Inode,
    kill_priv: bool,
    flags: u32,
) -> io::Result<(Option<Handle>, OpenOptions)> {
    self.do_open(inode, kill_priv, flags)
}

// AFTER
fn open(
    &self,
    ctx: Context,  // NOW USED
    inode: Inode,
    kill_priv: bool,
    flags: u32,
) -> io::Result<(Option<Handle>, OpenOptions)> {
    // set credentials before opening file for NFS root_squash compatibility
    let _credentials_guard =
        self.unix_credentials_guard(&ctx, &Extensions::default())?;
    self.do_open(inode, kill_priv, flags)
}
```

## How unix_credentials_guard Works

Located in `src/passthrough/credentials.rs`:

1. Gets current euid/egid
2. Calls `seteffgid()` then `seteffuid()` to switch to guest user's credentials
3. Returns a guard that restores original credentials on drop
4. The filesystem operation executes with guest user's fsuid/fsgid

```rust
// credentials.rs:37-85
pub fn set(self) -> io::Result<Option<UnixCredentialsGuard>> {
    let current_uid = HostUid::from(unsafe { libc::geteuid() });
    let current_gid = HostGid::from(unsafe { libc::getegid() });

    let change_uid = !self.uid.is_root() && self.uid != current_uid;
    let change_gid = !self.gid.is_root() && self.gid != current_gid;

    if change_gid {
        oslib::seteffgid(self.gid)?;
    }
    if change_uid {
        oslib::seteffuid(self.uid)?;
    }
    // ... returns guard that restores on drop
}
```

## Building

Use the provided Dockerfile to build for Debian bookworm (matches DO node OS):

```bash
docker build --platform linux/amd64 -t virtiofsd-builder -f Dockerfile.virtiofsd .
docker run --rm -v $(pwd)/output:/dest virtiofsd-builder
```

Output: `output/virtiofsd` (plus bundled libraries)

## Deployment

1. Copy binary to node: `/opt/kata/libexec/virtiofsd`
2. Rename via `mv` (can't overwrite running binary)
3. New pods will use the patched version

```bash
# via debug pod with host filesystem access
nsenter -t 1 -m -- mv /opt/kata/libexec/virtiofsd.new /opt/kata/libexec/virtiofsd
```

## Testing

After deploying, restart a kata pod with NFS storage and verify:

```bash
# inside kata container as uid=999
echo "test" > /workspace/test.txt
ls -la /workspace/test.txt
# should show: -rw-r--r-- 1 nfsb nfsb ... (not nobody nogroup)

# all operations should work
echo "append" >> /workspace/test.txt  # append
echo "new" > /workspace/test.txt      # overwrite
rm /workspace/test.txt                # delete
mkdir /workspace/dir && rmdir /workspace/dir  # directory ops
```

## Related Issues

- **GitLab #201** - NFS file handle stale errors (different issue)
  - About FUSE not supporting persistent file handles required by NFS
  - Requires kernel FUSE changes, being tracked by maintainer
  - Not related to this credentials fix

## Files

- `nfs-credentials-fix.patch` - The source patch
- `Dockerfile.virtiofsd` - Build configuration
- `src/passthrough/mod.rs` - Patched file
- `src/passthrough/credentials.rs` - Credentials implementation (read-only reference)
