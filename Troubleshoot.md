# Troubleshooting Sintinel

## Problem

**Error:** Repository is already locked by another process.

While running the `sintinel remove` command, the following error is observed:

```
repo already locked, waiting up to 0s for the lock
unable to create lock in backend: repository is already locked by PID 34 on e6c74f26c33b by sintinel (UID 1000, GID 1000)
lock was created at 2025-10-26 07:46:54 (11h32m41.00453619s ago)
storage ID 405aaee2
the `unlock` command can be used to remove stale locks
2025-11-08 07:19:35 [ERROR] Command failed: restic forget --prune a7e0113b
2025-11-08 07:19:35 [ERROR] ❌ Command failed: restic forget --prune a7e0113b
```

## Reason

Restic uses locks to prevent concurrent write operations that could corrupt the repository. Another process started **Sintinel** earlier, and the lock it created is preventing the new `restic forget --prune` command from running.

This usually happens when:

* The original process crashed, or
* The process exited abnormally and never released the lock.

Restic will block `forget` or `prune` operations until the lock is cleared.

## Resolution

### 1. Verify No Other Restic Process Is Running

Check whether the process ID reported in the error message is still active:

```
ps -p 34
```

If no output is returned, the process is no longer running and the lock is considered **stale**.

### 2. Manually Unlock the Repository

If the lock is confirmed to be stale, unlock the repository manually:

```
restic -r /path/to/repo/url unlock
```

**Example:**

```
restic -r rest:http://vop-customerzabbix-db-user:XXXXXXXXXX@localhost:8000/vop-customerzabbix-db-user unlock
```

You will be prompted for the repository password.

Replace `/path/to/repo/url` with the correct repository URL for your environment. This command removes the stale lock.

## Notes

* **Only run `restic unlock` if you are absolutely certain that no other backup or Restic process is running.** Removing an active lock can result in repository corruption.
* Repository credentials are stored securely in **PasswordManager**.
