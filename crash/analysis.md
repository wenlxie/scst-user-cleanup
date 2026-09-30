# Original SCST cleanup panic: `crash` dump analysis

This document records the analysis of the **original x86_64 production dump**,
`dump.202609291605`. It distinguishes values read from the dump from the
cleanup sequence inferred from those values and the SCST source. The separate
Lima VM reproduction and before/after fix results are in
[validation.md](../repro/validation.md).

## Inputs and tool version

The dump and matching `vmlinux` are on a separate analysis VM at
`/home/centos/dump.202609291605` and `/home/centos/vmlinux`. They are from a
different host and kernel than the Lima reproduction VM. The dump and
`vmlinux` are not stored in this repository.

The first attempt with `crash 8.0.0` displayed a kernel-version-inconsistency
warning and stopped during slab-cache initialization with:

```text
crash: invalid structure member offset: kmem_cache_s_num
       FILE: memory.c  LINE: 9619  FUNCTION: kmem_cache_init()
```

The analysis below was rerun on 2026-09-30 with `crash 9.0.3`, which opened the dump and permitted
stack, list, memory, and slab inspection. The dump is partial, so the analysis
only relies on objects that `crash` could read. The failed `crash 8.0.0`
session is a debugger startup failure; it is distinct from the SCST kernel
panic recorded in the dump.

Start the debugger on the analysis VM:

```sh
crash -s /home/centos/vmlinux /home/centos/dump.202609291605 \
  < crash.commands > crash.output.txt
```

## 1. Identify the panic and the active threads

```text
crash> sys
crash> bt
crash> bt -f 4263
crash> bt 414363
```

`sys` reports `RELEASE: 6.8.0-130-generic`, marks the input as `[PARTIAL
DUMP]`, and prints:

```text
PANIC: "Kernel panic - not syncing: SCST panic: DeadLoop error occurred!"
```

The panic stack is:

```text
PID: 4263  COMMAND: "scst_usr_cleanu"
panic
dev_user_cleanup_thread [scst_user]
kthread
```

The full stack (`bt -f 4263`) contains `ffff8daf894a2b80`, the SCST user
device being processed, and `ffff8daf894a3288`, its cleanup-list entry. Call
that device **A**. PID 414363 (`scst_usr_releas`) is waiting in
`msleep -> scst_acg_del_lun -> scst_unregister_virtual_device ->
dev_user_exit_dev -> __dev_user_release`.

These stacks directly establish that the cleanup worker called the panic while
device release was in progress. The original version of
`dev_user_process_cleanup()` had a `loop_count > 10000` branch with this exact
panic string. In the committed fix, that branch is replaced by yielding the
current device to the outer cleanup worker.

## 2. Inspect the device cleanup queue

```text
crash> list -H ffffffffc1fd23a0
```

The list contains `ffff8db0cea44108`, the cleanup-list entry of another
device, **B** (`ffff8db0cea43a00`). It does not contain A's entry
`ffff8daf894a3288`. That is consistent with the source's
`dev_user_cleanup_thread()` flow: remove A from the queue, call
`dev_user_process_cleanup(A)`, then proceed to the next device only after the
call returns. At the panic, the worker was still processing A.

Do not use `p cleanup_list` to decide whether the queue is empty in this
session: it printed `{ first = 0x0 }`, which conflicts with the direct list
traversal. The list head's raw memory and `list -H` output are the useful
checks here.

## 3. Inspect A's retained user command

```text
crash> rd -64 ffff8d6c8f33dd80 16
crash> kmem ffff8d6c8f33dd80
```

The relevant raw words are:

```text
ffff8d6c8f33dd80: 0000000000000000 ffff8daf894a2b80
ffff8d6c8f33dd90: 0000000000000003 ffff8d6c8f33dd80
ffff8d6c8f33dda0: 0000002000000001 0000000000000020
ffff8d6c8f33ddb0: 000058f76cf2e000 ffff8d2c31436600
ffff8d6c8f33ddc0: 0000000000000000 0000000700000008
```

Decode these using `struct scst_user_cmd` in upstream
[`scst_user.c`](https://github.com/SCST-project/scst/blob/5908a8da46c3f98dc1524d31d761bd4dd642b151/scst/src/dev_handlers/scst_user.c)
and the state constants in
[`scst_user.h`](https://github.com/SCST-project/scst/blob/5908a8da46c3f98dc1524d31d761bd4dd642b151/scst/include/scst_user.h):

| Offset | Field | Observed value |
| --- | --- | --- |
| `+0x00` | `cmd` | `NULL` |
| `+0x08` | `dev` | `ffff8daf894a2b80` (A) |
| `+0x18` | `buf_ucmd` | `ffff8d6c8f33dd80` (itself) |
| `+0x20` | `ucmd_ref` (low 32 bits) | `1` |
| `+0x30` | `ubuff` | `000058f76cf2e000` |
| `+0x40` | `sgv` | `NULL` |
| `+0x48` | flags | `0x8`: `sent_to_user=0`, `seen_by_user=1` |
| `+0x4c` | `state` | `7`: `UCMD_STATE_ON_FREE_SKIPPED` |

`kmem` reports `[ffff8d6c8f33dd80]` as an **allocated** `scst_user_cmd`
slab object. A's SCST command is gone, but its user command still has one
reference. The source shows that `dev_user_unjam_dev()` counts commands in the
hash before skipping entries with `sent_to_user=0`; such an entry can therefore
keep its result nonzero while `dev_user_get_next_cmd()` has no ready command.

## 4. Follow the SGV buffer to B

Read the SGV object and the second user command:

```text
crash> rd -64 ffff8d6d8107f980 12
crash> kmem ffff8d6d8107f980
crash> rd -64 ffff8d6d90b3bc80 16
crash> kmem ffff8d6d90b3bc80
```

The decisive SGV word is:

```text
ffff8d6d8107f9c0: 0000000000000020 ffff8d6c8f33dd80
```

At offset `+0x48`, `struct sgv_pool_obj.allocator_priv` points to A's user
command. `kmem` reports this SGV as an allocated `sgv-128K` slab object.
`sgv_get_priv()` returns `allocator_priv`; `dev_user_alloc_sg()` sets the
consumer's `buf_ucmd` from that value.

The relevant words in B's command are:

```text
ffff8d6d90b3bc80: ffff8d7118b07900 ffff8db0cea43a00
ffff8d6d90b3bc90: 0000000000000001 ffff8d6c8f33dd80
ffff8d6d90b3bcb0: 000058f76cf2e000 0000000000000000
ffff8d6d90b3bcc0: ffff8d6d8107f980 0000000300000009
```

| Field | Observed value |
| --- | --- |
| `cmd` | `ffff8d7118b07900` (non-NULL) |
| `dev` | `ffff8db0cea43a00` (B) |
| `buf_ucmd` | `ffff8d6c8f33dd80` (A's command) |
| `ubuff` | `000058f76cf2e000` (same as A) |
| `sgv` | `ffff8d6d8107f980` |
| `sent_to_user`, `state` | `1`, `3` (`UCMD_STATE_EXECING`) |

`kmem` also reports B's command as an allocated `scst_user_cmd` object. Thus
the dump directly shows B holding an SGV object whose allocation owner is A's
retained command.

Read the two device identifiers to check that A and B are different devices:

```text
crash> rd -a ffff8daf894a3250 56
crash> rd -a ffff8db0cea440d0 56
```

Their identifiers end in `cb958dd30766` and `5c233f6618b1`, respectively.
The shorter `rd -a ... 4` command from the first batch only printed the common
prefix and was insufficient for this comparison.

## 5. Explain the dead loop and fix

The observed dependency is:

```text
cleanup worker processing A
  -> A has a retained command (ref=1, no ready work)
  -> B is using the SGV object whose allocator_priv is A's command
  -> B's cleanup entry is queued behind A
```

The source explains the resulting stall. `sgv_pool_flush()` can release SGV
objects returned to the pool cache, but B's object is still checked out. The
original inner loop in `dev_user_process_cleanup(A)` kept retrying while A's
hash still contained a command and no ready command was available. It never
returned to the outer worker to handle B, and eventually reached the explicit
`DeadLoop` panic. The checked-out-object and worker-dependency explanation is
an **inference from the dump state plus source control flow**; it is not a
recorded timeline in the dump.

The fix yields A when there are no ready commands but retained commands remain.
The outer worker can then process B and retry A later. The existing post-unjam
SGV flush from upstream PR #378 only handles objects that have entered the
pool cache; it cannot release B's checked-out object by itself. The Lima
reproduction in [validation.md](../repro/validation.md) produced the same
cleanup stall and soft lockup with original upstream code, including PR #378,
and completed cleanup with this fix. The unmodified reproduction did not
produce this locally added panic message.

## Evidence and limits

- The production panic was investigated with `crash`, including `sys`, `bt`,
  `list`, `rd`, and `kmem`. The Lima reproduction used backend and kernel
  console logs; it was not a second `crash` dump analysis.
- SCST private structure types were unavailable to `crash` in this session.
  `struct scst_user_cmd` and `struct sgv_pool_obj` could not be printed directly,
  so field offsets above were decoded from raw memory using the source layout.
- The dump is partial. The pointer chain and stacks above were readable; the
  entire earlier lifetime of each object cannot be reconstructed from it.
- The rerun's complete command list and output are in
  [crash.commands](crash.commands) and [crash.output.txt](crash.output.txt).
