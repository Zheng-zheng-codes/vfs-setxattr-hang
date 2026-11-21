# INFO: task hung in vfs_setxattr (fs/xattr.c:343, v7.4-rc3)

Syzkaller-found hang on Linux v7.4-rc3, reproduced by sequentially replaying
the crash-log tail on fresh VMs.  No state snapshot and no auxiliary
tooling is required.

## Summary

Six tasks blocked for more than 140 seconds.  Several `lsetxattr()` callers
wait for the inode write lock (`i_rwsem`) in `vfs_setxattr()`, while the
task that holds that lock is stuck waiting for XFS log space reservation:

```text
INFO: task syz.1.94:13368 blocked for more than 143 seconds.
task:syz.1.94        state:D
Call Trace:
 rwsem_down_write_slowpath
 down_write
 inode_lock include/linux/fs.h:1027 [inline]
 vfs_setxattr+0x138/0x3c0 fs/xattr.c:343
 do_setxattr fs/xattr.c:662 [inline]
 filename_setxattr fs/xattr.c:690 [inline]
 path_setxattrat fs/xattr.c:734 [inline]
 __x64_sys_lsetxattr
```

The lock holder's own trace shows where the chain stops:

```text
 xlog_grant_head_wait fs/xfs/xfs_log.c:217
 xlog_grant_head_check fs/xfs/xfs_log.c:279
 xfs_log_reserve fs/xfs/xfs_log.c:387
 xfs_trans_alloc fs/xfs/xfs_trans.c:260
 xfs_trans_alloc_inode fs/xfs/xfs_trans.c:1088
 xfs_attr_set fs/xfs/libxfs/xfs_attr.c:1183
 xfs_attr_change fs/xfs/xfs_xattr.c:128
```

i.e. an `xfs_attr_set()` (setxattr on XFS) task holds the inode lock while
waiting for log grant space in `xlog_grant_head_wait()`, and every other
setxattr on the same inode piles up behind `vfs_setxattr()`.

## Relation to the xlog_grant_head_check hang

A hang of `xlog_grant_head_check()` (fs/xfs/xfs_log.c:279) was observed
previously on the same tree family; this report is the same log-grant-space
wait seen from the `vfs_setxattr()` vantage point instead of the xfs log
path, with the additional detail that the waiter holds `i_rwsem` and
therefore blocks all concurrent xattr operations on that inode.

## Reproduction

  git clone https://github.com/Zheng-zheng-codes/vfs-setxattr-hang
  cd vfs-setxattr-hang

  # boot v7.4-rc3 with the supplied kernel config (Debian trixie x86_64)
  # in the guest, as root:
  syz-execprog -procs=1 -threaded -repeat=1 \
      -executor ./syz-executor reproducer/repro.kernsight.prog

The reproducer is the crash-log tail (160 programs in log format) replayed
sequentially; the hang appears during the replay itself.  Running it with
`-procs=8` did not reproduce the hang under our tests; sequential execution
is required.

## Verification

Stability verification on fresh VMs (fresh qcow2 overlay per run,
sequential replay):

- five independent runs: **5/5** reproductions, 181-205 s to the hung-task
  dump in every run (2026-11-21); see `external-verify/`

## Kernel / guest details

  kernel: v7.4-rc3, bzImage sha256
    0c16409f8aec508ac44a63e631c2bd352536401f3fd0856df1d839fb953a1882
  guest: Debian trixie x86_64, 6 GiB RAM, 4 vCPU, QEMU/KVM
  boot: oops=panic panic_on_warn=1 hung_task_panic=0
  config: CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y, CONFIG_XFS_FS=y
  syzkaller: commit 853b104bb454141a5b0fc0e4218ec78e9f34ffb1

See `environment.txt` for the full QEMU command line and image hashes.
